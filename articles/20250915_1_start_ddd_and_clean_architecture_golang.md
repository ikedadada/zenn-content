---
title: "DDDとクリーンアーキテクチャをはじめよう-Golang編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DDD", "クリーンアーキテクチャ", "golang", "echo", "gorm"]
published: true
---

## 背景

ども！池田(ikedadada)です！

前回はNode.js編でDDD＋クリーンアーキテクチャの設計と実装方針を一通りまとめました。本記事では、同じ要件のTodoAPIをGolang（Echo +
GORM + MySQL）で実装する際の「設計の勘所」と「Goならではのポイント」にフォーカスして紹介します。

リポジトリ（Go実装含む）:

https://github.com/ikedadada/start-ddd-and-clean-architecture

以降のサンプルは `backend_golang/` ディレクトリ配下の実装を前提にしています。

## 全体像と依存の向き

レイヤは以下の4層です。依存は内向きのみになります。

- presentation: Echoを使ったHTTPハンドラ、DTO、バリデーション、エラーマッピング
- application_service: ユースケース、トランザクション境界の確立
- domain: エンティティ（不変条件・状態遷移）、リポジトリのポート
- infrastructure: GORMやDB接続、データモデル、トランザクション実装（アダプタ）

Goではパッケージ分割と公開・非公開を使って境界を明確にします。特にドメインのエンティティはフィールドを未公開にして、状態遷移をメソッドに閉じ込めるのが要点です。

## ドメインモデル（Todo）

Todoは未公開フィールドで不変条件を守ります。生成は
`NewTodo`、更新は専用コマンド、完了状態の切替はメソッドで表現します。

```go
// backend_golang/domain/model/todo.go（抜粋）
type Todo struct {
    id          uuid.UUID
    title       string
    description *string
    completed   bool
}

func NewTodo(title string, description *string) Todo {
    return Todo{ id: newID(), title: title, description: description, completed: false }
}

func (t *Todo) MarkAsCompleted() error {
    if t.completed { return ErrTodoAlreadyCompleted }
    t.completed = true; return nil
}

func (t *Todo) MarkAsNotCompleted() error {
    if !t.completed { return ErrTodoNotCompleted }
    t.completed = false; return nil
}

type CommandUpdateTodo struct { Title string; Description *string }
func (c *CommandUpdateTodo) Update(t *Todo) { t.title = c.Title; t.description = c.Description }
```

ポイント。

- 未公開フィールドで直接変更を防ぎ、状態遷移はメソッド経由に限定。
- 完了状態の整合性は `MarkAsCompleted/MarkAsNotCompleted` が担保。
- 上書き更新は `CommandUpdateTodo` に責務を寄せ、`completed` を変更不可に。
- I/Oは `Serialize/Deserialize` で統一して境界を明確化。

## リポジトリ（ポート）

リポジトリはドメイン型を返します。存在なしはインフラのエラーを `repository.ErrRepositoryNotFound`
に正規化します。

```go
// backend_golang/domain/repository/todo_repository.go（抜粋）
type TodoRepository interface {
    Save(ctx context.Context, todo model.Todo) error
    FindByID(ctx context.Context, id uuid.UUID) (model.Todo, error)
    FindAll(ctx context.Context) ([]model.Todo, error)
    Delete(ctx context.Context, todo model.Todo) error
}
```

## ユースケース（アプリケーションサービス）

読み取り系はそのまま呼出、変更系は必ずトランザクション境界で実行します。入力はユースケース用のDTO、出力はドメインを返し、外側でDTOへ変換します。

```go
// backend_golang/application_service/usecase/update_todo_usecase.go（抜粋）
func (u *updateTodoUsecaseImpl) Handle(
    ctx context.Context,
    in UpdateTodoUsecaseInput,
) (todo model.Todo, err error) {
    err = u.tx.Run(ctx, func(ctx context.Context) error {
        todo, err = u.tr.FindByID(ctx, in.ID)
        if err != nil { return err }
        cmd := model.CommandUpdateTodo{ Title: in.Title, Description: in.Description }
        cmd.Update(&todo)
        return u.tr.Save(ctx, todo)
    })
    if err != nil { return model.Todo{}, err }
    return todo, nil
}
```

## TransactionServiceの実装（contextでTx伝播）

Goでは `context.Context` にTxを載せ、`DB.Conn(ctx)` がTx有無を判定して接続を返します。ユースケースは
`tx.Run(ctx, fn)` を使うだけでトランザクションが透過されます。

```go
// infrastructure/service/transaction_service.go（抜粋）
func (s *TransactionService) Run(
    ctx context.Context,
    fn func(ctx context.Context) error,
) (err error) {
    tx := s.db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
            err = fmt.Errorf("recovered panic: %v", r)
        }
    }()
    ctx = infrastructure.WithTx(ctx, tx)
    if err = fn(ctx); err != nil { tx.Rollback(); return }
    return tx.Commit().Error
}

// infrastructure/db.go（抜粋）
func (d *DB) Conn(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(txKey).(*gorm.DB); ok && tx != nil { return tx }
    return d.DB.Session(&gorm.Session{})
}
```

ポイント。

- トランザクション境界はユースケースに置き、内側はTxを意識しない。
- `panic` 回復とRollbackを `defer` で一括処理。
- `context` によるTxの透過で、関数引数にTxをばらまかない。

## インフラ（アダプタ）

GORM用のデータモデルはドメインと分離し、変換はアダプタの責務にします。

```go
// infrastructure/repository/data_model/todo.go（抜粋）
type Todo struct {
    ID string `gorm:"type:uuid;primaryKey"`
    Title string `gorm:"type:varchar(255);not null"`
    Description *string `gorm:"type:text"`
    Completed bool `gorm:"type:boolean;not null"`
}

func (t *Todo) ToModel() model.Todo { /* Serialize/Deserialize を利用 */ }
func FromModel(m model.Todo) Todo { /* 逆変換 */ }
```

存在なしは `gorm.ErrRecordNotFound` を捕捉してポート側の `ErrRepositoryNotFound` に変換。

```go
// infrastructure/repository/todo_repository.go（抜粋）
if err := conn.First(&dt, "id = ?", id).Error; err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return model.Todo{}, repository.ErrRepositoryNotFound
    }
    return model.Todo{}, err
}
```

## プレゼンテーション（Echo）

ハンドラは Bind → Validate → ユースケース → ドメイン→DTO変換 → JSON という流れ。エラーは
`repository.ErrRepositoryNotFound` を404へマッピングします。

```go
// presentation/handler/get_todo_handler.go（抜粋）
todo, err := h.gu.Handle(c.Request().Context(), req.ID)
if err != nil {
    if errors.Is(err, repository.ErrRepositoryNotFound) {
        return echo.NewHTTPError(http.StatusNotFound, "Todo not found")
    }
    return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
}
```

共通のエラーハンドラで `echo.HTTPError` をJSON整形し、バリデータは独自実装を設定します。

## ルーティング（仕様準拠）

- POST `/todos` 新規作成（201）
- GET `/todos` 全件取得（200）
- GET `/todos/:id` 1件取得（200/404）
- PUT `/todos/:id` 上書き更新（200/404）
- PUT `/todos/:id/complete` 完了（200/409/404）
- PUT `/todos/:id/uncomplete` 未完了に戻す（200/409/404）

## テスト

- ドメイン: 状態遷移と不変条件のテスト（完了/未完了の整合性）
- ユースケース: 正常系・異常系（NotFound、重複完了など）
- インフラ: リポジトリのCRUD、Txの結合テスト
- プレゼンテーション: ハンドラの入出力・エラーマッピングの確認

## 実装時の重要ポイント（Go視点）

- 未公開フィールドでエンティティの整合性を守る（コンストラクタ必須）。
- ドメインに副作用や入出力の詳細を持ち込まない（Serializeで境界）。
- ユースケースがトランザクション境界を持ち、`context` でTxを透過。
- エラーは段階的にマッピング（GORM→ポート→HTTP）。`errors.Is` を活用。
- DTO変換はプレゼンテーション層で行い、ドメイン型はそのまま返す。
- インターフェースは最小に保つ（`TodoRepository` はCRUD最小セット）。
- `uuid.NewV7()` のような生成はドメインの中で閉じ、外から差し込まない。

## 動かしてみる

Docker ComposeでDBとGoサーバを起動できます。

```bash
docker compose up -d --build db golang
curl localhost:3001/health
```

POST例:

```bash
curl -X POST localhost:3001/todos \
  -H 'Content-Type: application/json' \
  -d '{"title":"Write Go article","description":"DDD + Clean Architecture"}'
```

## まとめ

GoでDDD＋クリーンアーキテクチャを実践する際は、未公開フィールドとメソッドでドメインの整合性を守り、ユースケースでトランザクション境界を握るのが肝です。
`context`
によるTx伝播、エラーの段階的マッピング、DTO変換の責務分離により、「内向き依存」「詳細の交換可能性」「テスト容易性」を自然に満たせます。

次は必要に応じてマイグレーションや観測（ログ/メトリクス）を足していきましょう。
