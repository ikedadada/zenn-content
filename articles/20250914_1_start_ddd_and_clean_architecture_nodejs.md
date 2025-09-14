---
title: "DDDとクリーンアーキテクチャをはじめよう-Node.js編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  [
    "DDD",
    "クリーンアーキテクチャ",
    "ソフトウェア設計",
    "アーキテクチャ",
    "設計原則",
    "Node.js",
    "Express",
    "TypeScript",
  ]
published: false
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

## リポジトリ（ポート）

ファイル: `src/domain/repository/todoRepository.ts`

- ドメインはインターフェースのみを定義する。
- `findAll`/`findById`/`save`/`delete` の基本操作だけを提供する。
- 見つからない場合は `RepositoryNotFoundError` として扱う。

:::message

「ドメインが技術詳細を知らない」ために、PrismaやSQLについては記載しないことを徹底します。

:::

## ユースケース（アプリケーションサービス）

場所: `src/application_service/usecase/*`

- 読み取り系はリポジトリから取得して返すだけ。
- 書き込み系は `TransactionService` でトランザクション境界を張る。
- `update` では `completed` を変更しない。完了/未完は専用ユースケースに分離する。

例: `updateTodoUsecase.ts` はタイトルと説明だけを更新します。例: `markAsCompletedTodoUsecase.ts`
はエンティティの `markAsCompleted` を呼び、保存します。

これにより「完了フラグの一貫性は常にユースケースを経由して担保する」という方針が徹底できます。

## インフラ（アダプタ）

場所: `src/infrastructure`

- `repository/todoRepository.ts` はPrismaでCRUDを実装する。
  - ドメインとデータモデルのマッピングは `toDatamodel`/`fromDatamodel` で明示する。
- `repository/context.ts` は `AsyncLocalStorage`
  を使い、トランザクション中に同じ接続を透過的に共有する（`ALSContextProvider`）。
- `service/transactionService.ts` は Prisma の `$transaction`
  で関数を実行し、その間だけ ALS にトランザクションクライアントを注入する。

結果として、ユースケースは「接続のことを知らずに」一連の操作を安全に実行できます。

## プレゼンテーション（Web層）

場所: `src/presentation`

- `handler/*.ts` は Express 5 のハンドラである。入力バリデーションは `zod` を使う。
- 例外はミドルウェアでHTTPエラーにまとめて変換する。
  - `errorHandler.ts` は 400/404/409/500 などを返す。
  - `routeNotFoundHandler.ts` は未定義ルートを404にする。

ハンドラは「入出力の整形」に集中し、ビジネスルールはユースケースとドメインに任せます。

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

## まとめ

Node.jsでDDD＋クリーンアーキテクチャを実践するなら、「内向き依存」「ユースケース中心」「詳細の交換可能性」を軸に設計するのが心地よいです。今回のサンプルでは、PrismaやExpressといった具体的な技術は外側に寄せ、ドメインとユースケースはシンプルに保ちました。

次回は、実装の中でもつまずきやすいポイントや、拡張時の分岐（認可や監査など）の置き場、運用を見据えた構成の工夫についても触れていきます。お楽しみに！
