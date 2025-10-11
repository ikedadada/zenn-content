---
title: "DDDとクリーンアーキテクチャをはじめよう-Rust編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DDD", "クリーンアーキテクチャ", "rust", "axum", "sqlx"]
published: true
---

## 背景

ども！池田(ikedadada)です！

シリーズもいよいよRust編です。これまでNode.js、Go、Python、Javaの実装を紹介してきましたが、今回は同じTodo
APIの要件をRust（Axum + SQLx + MySQL）で実装するときの構成とキーポイントを整理します。

ソースコード:

https://github.com/ikedadada/start-ddd-and-clean-architecture/tree/main/backend_rust

:::message

Rustは所有権と非同期ランタイムが独特なので、「トランザクション境界」と「コンテキスト共有」をどう表現するかに特に気を配りました。シリーズ共通のレイヤ構造を崩さずにRustらしく落とし込む方法を中心に紹介します。

:::

## Rust版の全体像

レイヤ構成は他言語編と同じです。依存は常に内向きで、外側の技術的詳細を中へ漏らしません。

- `domain`: エンティティとリポジトリポート
- `application`: ユースケースとトランザクションサービス
- `infrastructure`: SQLxによるアダプタ、接続コンテキスト
- `presentation`: Axumハンドラ、ミドルウェア、エラーマッピング
- `main.rs`: Composition Root（依存解決とルーティング）

Rustでは `Arc`
とトレイトオブジェクトで依存を束ね、Tokioの非同期実行と組み合わせて各層を接続します。シリーズでおなじみのエンドポイント（GET/POST/PUT/DELETEと完了フラグ操作）もそのまま再現しています。

## ドメインモデル：所有権で不変条件を守る

ファイル: `backend_rust/src/domain/model/todo.rs`

- `Todo` はUUID v7でIDを生成し、完了フラグは `mark_as_completed` / `mark_as_uncompleted` で制御。
- `TodoDto` を定義して永続化層（SQLx）との境界を明示。`From` 実装でDTO ⇔ ドメインを往復。
- テスト（同ファイル内）で状態遷移とDTOラウンドトリップを検証。

```rust
#[derive(Debug, Clone)]
pub struct Todo {
    id: Uuid,
    title: String,
    description: Option<String>,
    completed: bool,
}

impl Todo {
    pub fn new<T: Into<String>>(title: T, description: Option<String>) -> Self {
        Self { id: Uuid::now_v7(), title: title.into(), description, completed: false }
    }

    pub fn mark_as_completed(&mut self) -> Result<(), TodoAlreadyCompletedError> {
        if self.completed {
            return Err(TodoAlreadyCompletedError);
        }
        self.completed = true;
        Ok(())
    }
}
```

所有権モデルのおかげで、完了状態はメソッド経由でのみ変更できます。可変参照を握った瞬間だけ状態が変わるため、GoやJavaでは「公開フィールドへ直接触れさせない工夫」を追加する必要がありません。

## リポジトリ：SQLx + task-local接続

ファイル: `backend_rust/src/infrastructure/repository/todo_repository.rs`

- `TodoRepositoryImpl` はSQLxの`MySqlConnection`を利用。非同期関数内で `CONNECTION_SLOT`
  から接続を取得。
- CRUDはユースケースと同じI/Oを返し、MySQL依存（`ON DUPLICATE KEY UPDATE` など）はここに閉じる。
- 例外は `RepositoryError::NotFound`/`DataAccess` に正規化し、アプリ層へ伝える。

Rustの非同期関数は暗黙にアドホックなスレッドへ移るので、`tokio::task_local!`
を使って「同じ非同期タスク内なら同じ接続を共有する」仕組みを用意しました（詳細は次節）。

## ContextProvider：Tokioのタスク局所ストレージで接続を共有

ファイル: `backend_rust/src/infrastructure/repository/context_provider.rs`

- `CONNECTION_SLOT`（task-local）に `Arc<Mutex<PoolConnection>>`
  を保持し、ネスト時も同一接続を再利用。
- `run_scoped` がスコープを張り、接続が無ければプールから取得してスロットへ格納。
- スコープ内では `connection()` を呼ぶだけで同じ接続を再取得できる。

```rust
tokio::task_local! {
    static CONNECTION_SLOT: ConnectionSlot;
}

pub async fn run_scoped<F, Fut, T>(&self, run: F) -> Result<T, ContextError>
where
    F: FnOnce() -> Fut + Send,
    Fut: Future<Output = Result<T, ContextError>> + Send,
{
    if CONNECTION_SLOT.try_with(|_| ()).is_ok() {
        return run().await;
    }

    let connection = self.pool.acquire().await?;
    let connection = Arc::new(Mutex::new(connection));

    CONNECTION_SLOT.scope(ConnectionSlot { connection }, async move { run().await }).await
}
```

Node.js編の`AsyncLocalStorage`、Python編の`ContextVar`と役割は同じです。Tokioタスク単位で接続を束ねることで、「ユースケースは接続やトランザクションを意識しない」まま処理を進められます。

## TransactionService：BEGIN/COMMIT/ROLLBACKを自前で発行

ファイル: `backend_rust/src/infrastructure/service/transaction_service.rs`

- `TransactionServiceImpl` が `BEGIN` → ユースケース → `COMMIT/ROLLBACK` を明示的に実行。
- `ContextProvider` から都度接続を取り出し、確実に同一接続でトランザクションを閉じる。
- ユースケース側は `transaction_service.run(|| async move { ... })` を呼ぶだけで境界を確立。

SQLxには専用のトランザクション型がありますが、task-local接続と組み合わせるために手動で制御しています。これにより、複数リポジトリや補助クエリを挟む場合でも同一トランザクションを共有できます。

## アプリケーションサービス：ユースケースはシンプルに

ファイル: `backend_rust/src/application/usecase/`

- 読み取り系（`GetAll`, `Get`）はリポジトリを呼んでDTOへ変換。
- 書き込み系（`Update`, `MarkAsCompleted`, `Delete` など）は `TransactionService::run`
  でトランザクション境界を設定。
- ドメイン例外（`TodoAlreadyCompletedError`など）は `UsecaseError::conflict_from_*` でHTTP
  409へマッピング。

インメモリのテスト用リポジトリとNoopトランザクションを `mod.rs` の `test_support`
に用意し、ユースケースのユニットテストからインフラ依存を切り離しています。

## プレゼンテーション層：Axum + validator + 独自エラーマッピング

- ルータ: `backend_rust/src/presentation/router/todo_router.rs`
- ハンドラ: `backend_rust/src/presentation/handler/` 以下
- ミドルウェア: `ValidatedJson` / `ValidatedPath`（`middleware/validate.rs`）
- エラー: `presentation/error.rs`

Axumの`FromRequest`を実装して `ValidatedJson<T>` を作成し、`validator`
クレートで入力値を検証します。UUIDは共通関数 `validate_uuid` を通し、失敗した場合は
`AppError::BadRequest` に変換。

ハンドラは「入力変換 → ユースケース実行 → ドメイン → DTO化 → JSON応答」に責務を限定。エラーは
`UsecaseError` を `AppError` へ変換し、最終的にHTTPステータスとJSONボディを返します。

```rust
pub async fn handle(
    &self,
    ValidatedPath(path): ValidatedPath<MarkAsCompletedPath>,
) -> Result<Json<TodoResponse>, AppError> {
    let input = MarkAsCompletedTodoInput { id: path.id.into() };
    let result = self.usecase.execute(input).await?;
    Ok(Json(TodoResponse::from(result.todo)))
}
```

## テスト戦略

- ドメイン: 所有権による状態遷移制御とDTOラウンドトリップをテスト。
- アプリケーション: インメモリ実装 + Noopトランザクションでユースケースを検証。
- インフラ: `infrastructure/test_support/mysql.rs`
  がTestcontainersのMySQLを起動し、SQLxの実挙動を確認。
- プレゼンテーション: Axumハンドラのテストはリクエスト生成→レスポンス検証でHTTP仕様を担保。

Tokioテスト（`#[tokio::test]`）を多用し、非同期処理やトランザクションが期待通りに動くかを実際のMySQLで確認しています。

## Rustならではの実装ポイント

- **所有権と可変参照**: ドメインモデルが独自に状態を守り、外側からの破壊的更新を防げる。
- **Arc + trait object**: Composition Rootでは `Arc<dyn TodoRepository>`
  のようにDIを行い、クレート境界を明確化。
- **task-local接続**:
  Tokioの`task_local!`で非同期タスク単位のコンテキストを再現し、他言語編で紹介したALS/ContextVarと同じ体験を提供。
- **明示的トランザクション制御**:
  SQLxの抽象よりも手動制御を選択し、BEGIN/COMMIT/ROLLBACKの発行タイミングをユースケース側で握る。
- **型システムによるバリデーション**: DTOとリクエスト型を分離し、`serde` + `validator`
  でHTTP層の検証を完結させる。

## 動かしてみる

Rust版だけで動かす場合は次の手順です（MySQLはDocker Composeのサービスを利用）。

```bash
# 依存セットアップ（miseでRustツールチェインを揃える）
cd backend_rust
mise install

# 環境変数を設定して起動
export DATABASE_URL="mysql://user:password@localhost:3306/todo"
cargo sqlx migrate run
cargo run
```

Docker Compose全体で動かす場合はリポジトリ直下で `docker compose up -d db rust` を実行すればOKです。

## まとめ

- 接続の伝播はTokioのtask-localで実現し、他言語編と同じ「ユースケースは接続を知らない」構造を維持。
- トランザクション境界は`TransactionService`が明示的に握り、BEGIN/COMMIT/ROLLBACKを手動で扱う。
- Axumとvalidatorの組み合わせでHTTP層のバリデーションとエラーマッピングを整理。
- 所有権・型システムのおかげでドメインの不変条件を自然に表現でき、テストもユニットと実DBで段階的に担保。

所有権や非同期実行といったRust固有の特性が影響するぶん難易度も上がりますが、DDD＋クリーンアーキテクチャの考え方もRustでそのまま通用すると感じています。
