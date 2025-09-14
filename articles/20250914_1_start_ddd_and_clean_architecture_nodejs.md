---
title: "DDDとクリーンアーキテクチャをはじめよう-Node.js編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DDD", "クリーンアーキテクチャ", "nodejs", "express", "typeScript"]
published: true
---

## 背景

ども！池田(ikedadada)です！

前回はDDDとクリーンアーキテクチャの考え方と、Todo
APIの要件をまとめました。今回は「Node.jsでDDD＋クリーンアーキテクチャをやるなら、こんな感じで組むよ」という方針で、実装例を紹介します。

サンプル実装は次のリポジトリに置いてあります。

https://github.com/ikedadada/start-ddd-and-clean-architecture

:::message

目的は「原則の実感」と「読みやすい分割」です。特定のFWやORMにロックインしないことを重視し、依存は内向きに揃えます。

:::

## 全体像と依存の向き

構成はシンプルなレイヤ構造です。依存は常に内向きです。

- `src/domain` ← もっとも内側。ビジネスルールを表現する。
- `src/application_service` ← ユースケース（アプリケーションサービス）を置く。
- `src/infrastructure` ← DBなどの技術的詳細（外側）を扱う。
- `src/presentation` ← Web層を担う（Expressのハンドラやミドルウェア）。
- `src/main.ts` ← Composition Root。依存を束ねてルーティングを設定する。

「ドメインは技術詳細を知らない」「ユースケースはドメインを呼ぶ」「外側は内側のインターフェースに従う」。この関係を崩さないことで、交換可能性とテスト容易性を確保します。

## ドメインモデル（Todo）

ファイル: `src/domain/model/todo.ts`

- フィールドは `id`, `title`, `description`, `isCompleted`。
- 完了状態の変更は `markAsCompleted` と `markAsNotCompleted` に限定する。
- 不正な状態遷移は `DomainConflictError` を投げる。
- `toPrimitives`/`fromPrimitives` で永続化との境界を明示する。

ポイントは「完了フラグの整合性はエンティティで守る」です。ユースケースやDBの都合で直接フラグをいじらないように、APIを絞っています。

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/domain/model/todo.ts

## リポジトリ（ポート）

ファイル: `src/domain/repository/todoRepository.ts`

- ドメインはインターフェースのみを定義する。
- `findAll`/`findById`/`save`/`delete` の基本操作だけを提供する。
- 見つからない場合は `RepositoryNotFoundError` として扱う。

:::message

「ドメインが技術詳細を知らない」ために、PrismaやSQLについては記載しないことを徹底します。

:::

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/domain/repository/todoRepository.ts

## ユースケース（アプリケーションサービス）

場所: `src/application_service/usecase/*`

- 読み取り系はリポジトリから取得して返すだけ。
- 書き込み系は `TransactionService` でトランザクション境界を張る。
- `update` では `completed` を変更しない。完了/未完は専用ユースケースに分離する。

例: `updateTodoUsecase.ts` はタイトルと説明だけを更新します。例: `markAsCompletedTodoUsecase.ts`
はエンティティの `markAsCompleted` を呼び、保存します。

これにより「完了フラグの一貫性は常にユースケースを経由して担保する」という方針が徹底できます。

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/application_service/usecase/markAsCompletedTodoUsecase.ts

## TransactionServiceの実装

リポジトリに直接DB接続を持たせず、`TransactionService` と
`ContextProvider`（ALS）で接続を配布する構成としました。

狙い（なぜリポジトリに接続を持たせないのか）

- 取引の境界をユースケースで決めるためである。
- 複数リポジトリの操作を1トランザクションで束ねるためである。
- ドメイン/アプリケーションがPrismaなどの詳細に依存しないためである。
- テスト時に接続や永続化戦略を差し替えやすくするためである。

仕組み。

| 実装側                 | 役割                                                                    |
| ---------------------- | ----------------------------------------------------------------------- |
| ALSContextProvider     | `AsyncLocalStorage` で接続を保存し、リポジトリに透過的に伝播させる。    |
| TransactionServiceImpl | `prisma.$transaction` で関数を実行し、その間だけALSにTx接続を注入する。 |
| TodoRepositoryImpl     | `ALSContextProvider` から接続を取得し、クエリを実行する。               |

使い方（ユースケース側）

- 書き込み系ユースケースは `transactionService.run(async () => { ... })` で処理全体を包む。
- 読み取りは必要に応じてTxなしで行う方針とする。

- ALSコンテキスト:

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/infrastructure/repository/context.ts

- TransactionService実装:

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/infrastructure/service/transactionService.ts

- リポジトリアダプタ（接続取得）:

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/infrastructure/repository/todoRepository.ts

- ユースケース側の境界設定:

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/application_service/usecase/updateTodoUsecase.ts#L1-L80

:::message alert

- ALSは非同期コンテキストの継続に依存する。タイマーや外部イベントにまたがる場合はコンテキストが失われないよう配慮する。
- Prismaのトランザクションは長時間保持を避け、必要最小限の範囲に留める。
- 例外はTxをロールバックさせる。失敗の種別はレイヤごとに適切なエラー型へマッピングする。

:::

## インフラ（アダプタ）

場所: `src/infrastructure`

- `repository/todoRepository.ts` はPrismaでCRUDを実装する。
  - ドメインとデータモデルのマッピングは `toDatamodel`/`fromDatamodel` で明示する。
- `repository/context.ts` は `AsyncLocalStorage`
  を使い、トランザクション中に同じ接続を透過的に共有する（`ALSContextProvider`）。
- `service/transactionService.ts` は Prisma の `$transaction`
  で関数を実行し、その間だけ ALS にトランザクションクライアントを注入する。

結果として、ユースケースは「接続のことを知らずに」一連の操作を安全に実行できます。

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/infrastructure/repository/context.ts

## プレゼンテーション（Web層）

場所: `src/presentation`

- `handler/*.ts` は Express 5 のハンドラである。入力バリデーションは `zod` を使う。
- 例外はミドルウェアでHTTPエラーにまとめて変換する。
  - `errorHandler.ts` は 400/404/409/500 などを返す。
  - `routeNotFoundHandler.ts` は未定義ルートを404にする。

ハンドラは「入出力の整形」に集中し、ビジネスルールはユースケースとドメインに任せます。

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/presentation/handler/createTodoHandler.ts

## ルーティング（仕様準拠）

ファイル: `src/main.ts`

- 仕様通りにエンドポイントを定義する。
  - GET `/todos`
  - GET `/todos/:id`
  - POST `/todos`
  - PUT `/todos/:id`
  - DELETE `/todos/:id`
  - PUT `/todos/:id/complete`
  - PUT `/todos/:id/uncomplete`
- Composition Root として、PrismaClient → ContextProvider → Repository → TransactionService →
  Usecase → Handler の順に依存を束ねる。

https://github.com/ikedadada/start-ddd-and-clean-architecture/blob/3456531209a9a6c58a91112ce24aee70a151250d/backend_nodejs/src/main.ts

## テスト

- Vitest で各レイヤを分けてテストする。
  - ドメインモデルのふるまい
  - ユースケースのI/O
  - インフラのリポジトリ実装（Prisma）
  - ハンドラとミドルウェアのHTTP変換
- Prismaはテスト用スキーマ（`prisma/test.prisma`）を用意し、実行時に生成する。

:::message

レイヤごとにテスト対象を絞ると、失敗時に原因が特定しやすくなります。

:::

## 実装時の重要ポイント（Node.js視点）

### ユビキタス言語をコードに反映する

- 用語を揃え、ドメイン・ユースケース・ハンドラで同じ言葉を使う。
- 例: 完了フラグはドメインでは `isCompleted`、DBでは `completed` とし、マッピングで吸収する。

### エンティティが不変条件を守る

- 状態遷移APIを用意し、直接フィールドを書き換えないようにする。
- 例: `markAsCompleted`/`markAsNotCompleted` は矛盾時に例外を投げる。

### バリデーションの層を分ける

- 入力値の形や型はハンドラで `zod` により検証する。
- ビジネスルールの整合性はドメインとユースケースで担保する。

### トランザクション境界はユースケースに置く

- 書き込みユースケースは `TransactionService` で全体を包む。

### 接続やコンテキストは透過的に伝播させる

- `AsyncLocalStorage` によりトランザクション内の接続を共有し、ユースケースからは意識させない。

### リポジトリはドメインモデルを返す

- データモデルはアダプタ層に閉じ込め、`from/to` マッピングを明示する。

### DTO変換はプレゼンテーション層で行う

- ユースケースはドメインを返し、ハンドラでレスポンス形に直す。

### エラーは段階的にマッピングする

- ドメインの競合は `DomainConflictError`、永続化の未検出は `RepositoryNotFoundError` とする。
- ハンドラでHTTPエラー（409/404）に変換し、最後にミドルウェアでシリアライズする。

### 生成と副作用の責務を分ける

- ID生成はドメインで行い、新規作成時に副作用を分散させない。

### Composition Rootを明確にする

- 依存の組み立ては `main.ts` のみで行い、他レイヤにDIの詳細を漏らさない。

## まとめ

Node.jsでDDD＋クリーンアーキテクチャを実践するなら、「内向き依存」「ユースケース中心」「詳細の交換可能性」を軸に設計するのが心地よいです。今回のサンプルでは、PrismaやExpressといった具体的な技術は外側に寄せ、ドメインとユースケースはシンプルに保ちました。クリーンアーキテクチャの原則を実装する際にトランザクション管理の実装が課題になることが多いですが、`AsyncLocalStorage`を活用することで、Node.jsでも自然に境界を設定できます。

次回はTypeScript以外の言語での実装例を紹介します。お楽しみに！
