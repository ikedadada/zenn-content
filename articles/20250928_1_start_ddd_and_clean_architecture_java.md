---
title: "DDDとクリーンアーキテクチャをはじめよう-Java編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DDD", "クリーンアーキテクチャ", "java", "spring", "mysql"]
published: true
---

## 背景

ども！池田(ikedadada)です！前回はGolang編でDDD＋クリーンアーキテクチャの設計と実装方針をまとめました。本記事では、同じTodo
APIをJava（Spring Boot + JDBC + MySQL）で組むときのキーポイントを紹介します。

https://github.com/ikedadada/start-ddd-and-clean-architecture

:::message

Java編も「依存は内向き」「ユースケース中心」というシリーズの骨格はそのままです。言語やFWが抱える制約をどこに閉じ込めるかを意識しながら、Node.js編やGo編、Python編で紹介した工夫を比較できるようにまとめていきます。

:::

## 全体像と依存方向

構成は他言語編と同じ4層＋Composition Rootです。パッケージは `io.github.ikedadada.backend_java`
配下に整理しています。

- domain: `Todo`エンティティとリポジトリポート（インターフェース）を配置する。
- application_service: ユースケースとトランザクション境界を担うサービスを置く。
- infrastructure: Spring JDBCによるアダプタ実装とトランザクション実装をまとめる。
- presentation: RESTコントローラと例外マッピングを担当する。
- main: `BackendJavaApplication` がSpring Bootを起動する。

Spring
Bootに寄せがちな設定やDIは極力外側（infrastructure/presentation）に閉じ込め、ドメインとユースケースを純粋なJavaクラスとして保ちます。これはNode.js編の`AsyncLocalStorage`やGo編の
`context.Context`と同様に、「内側から技術詳細を見せない」ための工夫です。

## ドメインモデル（Todo）

`Todo`エンティティ（`domain/model/Todo.java`）はUUID
v7相当の時間ベースIDをコンストラクタで生成します。 `markAsCompleted` と `markAsNotCompleted`
が状態遷移を閉じ、矛盾時は `DomainException` を投げます。descriptionは `Optional<String>`
で保持し、`normalizeDescription` がnullや未指定を `Optional.empty()`
に整えることで永続化層とのやり取りを安定させます。DTOサブクラス経由で永続化層へ値を受け渡します。

```java
public class Todo {
    private static final TimeBasedEpochGenerator GENERATOR =
            Generators.timeBasedEpochGenerator();

    private UUID id;
    private String title;
    private Optional<String> description;
    private boolean completed;

    public Todo(String title, Optional<String> description) {
        this(GENERATOR.generate(), title, description, false);
    }

    public void markAsCompleted() {
        if (completed) {
            throw new DomainException.TodoAlreadyCompleted(this);
        }
        completed = true;
    }

    public void update(String title, Optional<String> description) {
        this.title = Objects.requireNonNull(title);
        this.description = normalizeDescription(description);
    }

    private static Optional<String> normalizeDescription(Optional<String> description) {
        return description != null ? description : Optional.empty();
    }
}
```

ポイントは以下の通りです。

- JavaでもNode.js編やGo編と同じく「完了フラグをメソッドに閉じる」ことで一貫性を守る。
- SpringのBeanやアノテーションは持ち込まず、プレーンなPOJOとしてテスト容易性を確保する。
- descriptionはOptionalに正規化し、nullや未指定の扱いを統一する。
- DTOを静的内部クラスに置き、`NamedParameterJdbcTemplate` との変換責務を明示する。

## リポジトリ（JDBCアダプタ）

ドメイン側のポートは `TodoRepository` で最小CRUDのみを定義し、存在しない場合は
`RepositoryException.TodoNotFound` を投げます。（`domain/repository/TodoRepository.java`）

アダプタは `NamedParameterJdbcTemplate`
を使った実装にまとめています。SQLでUUIDを文字列として扱い、取得時はDTO経由でドメインに戻す構成です。MySQL依存の`ON DUPLICATE KEY UPDATE`
はこの層に閉じ込め、ドメイン側へ漏らしません。

```java
@Component
public class TodoRepositoryImpl implements TodoRepository {
    private final NamedParameterJdbcTemplate jdbcTemplate;

    @Override
    public ArrayList<Todo> findAll() {
        return new ArrayList<>(jdbcTemplate.query(
                "SELECT id, title, description, completed FROM todos",
                (rs, rowNum) -> new Todo.DTO(
                        rs.getObject("id", UUID.class),
                        rs.getString("title"),
                        rs.getString("description"),
                        rs.getBoolean("completed"))
                        .toDomain()));
    }

    @Override
    public void save(Todo todo) {
        var sql = """
                INSERT INTO todos (id, title, description, completed)
                VALUES (:id, :title, :description, :completed)
                ON DUPLICATE KEY UPDATE
                    title = :title,
                    description = :description,
                    completed = :completed
                """;
        jdbcTemplate.update(sql, Map.of(
                "id", todo.getId().toString(),
                "title", todo.getTitle(),
                "description", todo.getDescription().orElse(null),
                "completed", todo.isCompleted()));
    }
}
```

- Node.js編のPrismaリポジトリ、Go編のGORMアダプタ、Python編のSQLAlchemyと同様に、ORM固有のクセはここで吸収する。
- 返り値は常にドメイン型にそろえる。
- 404相当の例外はポート側で正規化し、後段でHTTPステータスにマッピングする。

## TransactionServiceとトランザクション境界

Springでは `@Transactional`
を貼るのが簡単ですが、それではインフラの仕組みがユースケースに漏れてしまいます。そこでインターフェースを
`application_service/service/TransactionService.java` に用意し、実装を `TransactionTemplate`
で包みます。（`infrastructure/service/TransactionServiceImpl.java`）。

```java
public <T> T run(Supplier<T> block) {
    return transactionTemplate.execute(status -> block.get());
}
```

ユースケース側は `transactionService.run(() -> { ... })`
でクロージャを渡すだけで済みます。これによりNode.js編の`AsyncLocalStorage`、Go編の`context`、Python編の`ContextVar`と同じ発想を保てます。副作用の境界をユースケースに置き、テスト時はモックやフェイクへ差し替えられる構造になります。

## ユースケース（アプリケーションサービス）

ユースケース実装は `application_service/usecase` にまとめています。読み取り系（`GetTodoUsecaseImpl`
など）はリポジトリ呼び出しに徹します。トランザクションが必要なユースケースは担保すべき処理を
`TransactionService` に委譲します。

```java
@Component
public class MarkAsCompletedTodoUsecaseImpl
        implements MarkAsCompletedTodoUsecase {

    @Override
    public Todo handle(UUID id) {
        return transactionService.run(() -> {
            Todo todo = todoRepository.findById(id);
            todo.markAsCompleted();
            todoRepository.save(todo);
            return todo;
        });
    }
}
```

- `MarkAsCompletedTodoUsecaseImpl` は `todo.markAsCompleted()` を呼んでから `save` する。
  - 完了済みなら`DomainException` をそのまま投げ、プレゼンテーション層で409へ変換する。
- `CreateTodoUsecaseImpl` はトランザクション不要なので `new Todo(...)` で生成して保存する。
  - 完了フラグはドメイン側が初期値を管理する。
- `DeleteTodoUsecaseImpl` は存在確認と削除を1トランザクションにまとめる。

## プレゼンテーション層（REST + 例外ハンドラ）

`TodoController`（`presentation/controller/TodoController.java`）はDTO変換とHTTPマッピングに集中させています。

Javaの`Optional`はJacksonが扱いづらいため、レスポンスDTOではOptionalフィールドをそのまま公開します。

入力DTOには `@JsonSetter(nulls = Nulls.AS_EMPTY)` を付与して `null` を `Optional.empty()`
に変換し、Jakarta Validationでタイトル必須などの最低限チェックも行います。

```java
@PostMapping(value = "/todos", produces = MediaType.APPLICATION_JSON_VALUE)
@ResponseStatus(HttpStatus.CREATED)
public CreateTodoResponse createTodo(@RequestBody @Valid CreateTodoRequest request) {
    Todo todo = createTodoUsecase.handle(request.title, request.description);
    return new CreateTodoResponse(todo);
}

@PutMapping(value = "/todos/{id}/complete", produces = MediaType.APPLICATION_JSON_VALUE)
public MarkAsCompletedResponse markAsCompleted(@PathVariable("id") @Valid UUID id)
        throws HttpException.NotFound, HttpException.Conflict {
    try {
        Todo todo = markAsCompletedTodoUsecase.handle(id);
        return new MarkAsCompletedResponse(todo);
    } catch (DomainException.TodoAlreadyCompleted e) {
        throw new HttpException.Conflict(e.getMessage());
    } catch (RepositoryException.TodoNotFound e) {
        throw new HttpException.NotFound(e.getMessage());
    }
}
```

シリーズで恒例のHTTPエラーマッピングは `HttpException` ＋ `ExceptionHandlerMiddleware`
にまとめました。 `@RestControllerAdvice` が `DomainException` や `RepositoryException`
を拾い、404/409/500に整形します。Python編のミドルウェアやNode.js編のエラーハンドラと同じ構造です。

## テスト戦略

レイヤごとにJUnit/Testcontainersを組み合わせたテストを用意しています。

- domain: `TodoTest` で状態遷移、DTOラウンドトリップ、例外を検証する。
- application_service:
  Mockitoでリポジトリとトランザクションを差し替え、ユースケースの振る舞いを確認する。
- infrastructure: Testcontainers MySQLと `sql/init.sql`
  を使い、実際の方言でCRUDとトランザクションを検証する。
- presentation: `@WebMvcTest` と `MockMvc` でハンドラのレスポンスと例外マッピングを確認する。

プレゼンテーション層のテストでは `description: null` を送った際に `Optional.empty()`
へ補正されることも検証しています。

```java
@Test
void runRunnableRollsBackOnException() {
    int before = countRecords();

    assertThrows(RuntimeException.class, () -> transactionService.run(() -> {
        jdbcTemplate.update(
                "INSERT INTO transaction_test_records (id, note) VALUES (?, ?)",
                UUID.randomUUID().toString(),
                "Transaction rollback");
        throw new RuntimeException("boom");
    }));

    assertThat(countRecords()).isEqualTo(before);
}
```

Node.js編のVitest、Go編のtesting + Testcontainers、Python編のPytest +
Testcontainersと同じく、「ドメインは高速に」「インフラは実DBで」という住み分けを徹底しています。

## 実行・開発の補助ツール

`pom.xml` では Spring Web / JDBC /
Validation に加えて Testcontainers と Log4j2 を組み合わせています。 `java-uuid-generator`
を直接依存に入れてUUID v7を生成している点もJava特有のポイントです。

ローカルで動かす場合は `mise` が Java 21 と Maven 3.9 をインストールし、`Dockerfile.dev`
でホットリロード環境を用意する。環境変数（`DATABASE_URL`
など）はSpringの`application.properties`から参照するため、コンテナ構成やローカル実行を同じ形でそろえられる。

## Java編で意識したポイント（まとめ）

- Node.js／Go／Pythonと同じく、「ドメインは技術詳細を知らない」構造をSpringでも貫く。
- `@Transactional` に頼らず `TransactionService` 経由で境界を明示し、ユースケースに責務を置く。
- JDBC層にMySQL固有の処理を閉じ込め、DTO経由でドメインとのギャップを埋める。
- OptionalやJakarta Validationの扱いなど、Javaならではの細部はプレゼンテーション層で吸収する。
- TestcontainersでMySQLを立ち上げ、インフラの挙動を実DBで担保する。

JavaでもDDD＋クリーンアーキテクチャの考え方はそのまま通用します。言語特有の課題は内側へ漏らさず、外側の層に吸収させることでシリーズ全体の構造を一貫させられました。

次回はRust編を紹介します。お楽しみに！
