---
title: "DDDとクリーンアーキテクチャをはじめよう-Python編"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DDD", "クリーンアーキテクチャ", "python", "fastapi", "sqlalchemy"]
published: true
---

## 背景

ども！池田(ikedadada)です！前回はGolang編でDDD＋クリーンアーキテクチャの設計と実装方針をまとめました。本記事では、同じTodo
APIをPython（FastAPI + SQLAlchemy + MySQL）で組むときのキーポイントを紹介します。Node.js編の
`AsyncLocalStorage`、Go編の`context.Context`に対応するPython特有の制約、特にSQLAlchemyのセッション管理とトランザクション境界の扱いに注目してください。

実装リポジトリ:

https://github.com/ikedadada/start-ddd-and-clean-architecture

:::message

SQLAlchemyの制限の影響が大きいため、Python版はNode.js版やGo版と比べてやや複雑になっています。

とはいえ、ドメイン層の構造や依存方向は変わらないので、DDD＋クリーンアーキテクチャの本質的な部分は共通です。

:::

以降は`backend_python/`配下を前提に説明します。

## 全体像と依存方向

レイヤ構成は既存記事と同じく内向き依存を徹底しています。

- domain: エンティティ／ドメインサービス／リポジトリポート
- application_service: ユースケースとトランザクションサービス
- infrastructure: SQLAlchemyベースのアダプタ（リポジトリ実装・トランザクション実装）
- presentation: FastAPIのルータ、ハンドラ、ミドルウェア
- main: Composition Root。依存解決とミドルウェア登録を担う

Python版でも「ドメインは技術詳細を知らない」「外側が内側のインターフェースに従う」点を守りつつ、SQLAlchemyの制限をインフラ層に閉じ込めます。

## SQLAlchemyの制限とセッション管理

SQLAlchemyのORMセッションはスレッド/asyncセーフではなく、「同じリクエスト内で共有する」「リクエスト境界を出たら破棄する」必要があります。そこで`ContextProvider`インターフェースを用意し、実装側で`ContextVar`にセッションを束縛しています。ネストしたトランザクションは`Session.begin_nested()`で吸収し、`expire_on_commit=False`でコミット後もドメインオブジェクトの値をそのまま扱えるようにしています。

```python
# backend_python/src/todo_api/infrastructure/repository/context_provider.py
class ContextProviderImpl(ContextProvider[Session]):
    def __init__(self, engine: Engine) -> None:
        self._session_factory = sessionmaker(bind=engine, expire_on_commit=False)
        self._state: ContextVar[Session | None] = ContextVar("session_state", default=None)

    @contextmanager
    def transaction(self) -> Iterator[Session]:
        existing = self._state.get()
        if existing is not None:
            with existing.begin_nested():
                yield existing
            return

        session = self._session_factory()
        token = self._state.set(session)
        try:
            with session.begin():
                yield session
        finally:
            self._state.reset(token)
            session.close()
```

FastAPI側では`SessionMiddleware`が必ずセッションを張り、レスポンスが4xx/5xxなら`rollback()`を呼びます。これはエラーをハンドラでキャッチしてJSONに変換する場合も、最終的なコミットタイミングをフレームワークに握らせないためです。

```python
# backend_python/src/todo_api/presentation/middleware/session_middleware.py
class SessionMiddleware(BaseHTTPMiddleware):
    async def dispatch(
        self,
        request: Request,
        call_next: Callable[[Request], Awaitable[Response]],
    ) -> Response:
        with self._context_provider.transaction() as session:
            response = await call_next(request)
            if response.status_code >= 400 and session.in_transaction():
                session.rollback()
            return response
```

## ドメインモデル（Todo）

ドメイン層はPythonでも純粋なオブジェクトに徹し、ORMやFastAPIに依存しません。`Todo`エンティティはUUIDv7でIDを生成し、完了フラグの整合性をメソッドで守ります。SQLAlchemyとの橋渡しにはPydanticベースの`TodoDTO`を用意し、永続化層との変換を明示しました。

```python
# backend_python/src/todo_api/domain/model/todo.py
class Todo:
    def __init__(self, title: str, description: str | None = None):
        self._id: UUID7 = uuid7()
        self._title = title
        self._description = description
        self._completed = False

    def mark_as_completed(self):
        if self._completed:
            raise TodoAlreadyCompletedError()
        self._completed = True

    def mark_as_uncompleted(self):
        if not self._completed:
            raise TodoNotCompletedError()
        self._completed = False

    def to_dto(self) -> TodoDTO:
        return TodoDTO(
            id=str(self.id),
            title=self.title,
            description=self.description,
            completed=self.completed,
        )

    @classmethod
    def from_dto(cls, dto: TodoDTO) -> "Todo":
        todo = cls.__new__(cls)
        todo._id = parse_uuid7(dto.id)
        todo._title = dto.title
        todo._description = dto.description
        todo._completed = dto.completed
        return todo
```

`__new__`を使って`__init__`をバイパスしているのは、SQLAlchemyから再構築する際に新しいUUIDを生成しないための工夫です。

## リポジトリ実装（SQLAlchemyアダプタ）

リポジトリはセッションへ直接アクセスせず、常に`ContextProvider`経由で取得します。`session.merge()`を使っているのは、ドメインモデルがSQLAlchemyのトラッキング対象外であるためです。`merge`は「識別子が同じなら更新、なければINSERT」を実現し、`flush()`で即時にSQLを発行して整合性を担保します。

```python
# backend_python/src/todo_api/infrastructure/repository/todo_repository.py
class TodoRepositoryImpl(TodoRepository):
    def find_all(self) -> list[Todo]:
        session = self.context_provider.current()
        todos = session.scalars(select(TodoDataModel)).all()
        return [todo.to_domain() for todo in todos]

    def save(self, todo: Todo) -> None:
        session = self.context_provider.current()
        todo_data_model = TodoDataModel.from_domain(todo)
        session.merge(todo_data_model)
        session.flush()

    def delete(self, todo: Todo) -> None:
        session = self.context_provider.current()
        session.execute(delete(TodoDataModel).where(TodoDataModel.id == str(todo.id)))
```

存在しないリソースは`RepositoryNotFoundError`に正規化し、プレゼンテーション層でHTTP
404にマッピングします。

## トランザクション境界とユースケース

複数のリポジトリ操作を束ねるユースケースは`TransactionService`を受け取り、関数を`Run`でラップします。これにより、インフラ層のセッション制御を意識せずにトランザクション境界を確立できます。単一の書き込みしか行わない`Create`だけはセッションミドルウェアに任せ、余計な抽象を増やしていません。

```python
# backend_python/src/todo_api/application_service/usecase/update_todo_usecase.py
class UpdateTodoUsecaseImpl(UpdateTodoUsecase):
    def __init__(
        self,
        todo_repository: TodoRepository,
        transaction_service: TransactionService,
    ) -> None:
        self.todo_repository = todo_repository
        self.transaction_service = transaction_service

    def execute(self, input_dto: UpdateTodoUsecaseInput) -> UpdateTodoUsecaseOutput:
        def func() -> UpdateTodoUsecaseOutput:
            todo = self.todo_repository.find_by_id(input_dto.id)
            todo.update(title=input_dto.title, description=input_dto.description)
            self.todo_repository.save(todo)
            return UpdateTodoUsecaseOutput(todo=todo)

        return self.transaction_service.Run(func)
```

トランザクション実装側では`ContextProvider.transaction()`を呼び出すだけなので、依存方向は内向きのままです。

## プレゼンテーション層（FastAPI）

FastAPIのハンドラは入出力変換に集中し、ユースケースエラーをHTTPエラーへマッピングします。例として`UpdateTodoHandler`はRepository層の`RepositoryNotFoundError`を404に変換しています。

```python
# backend_python/src/todo_api/presentation/handler/update_todo_handler.py
class UpdateTodoHandler:
    def handle(self, id: UUID7, request: UpdateTodoRequest) -> UpdateTodoResponse:
        try:
            result = self.update_todo_usecase.execute(
                UpdateTodoUsecaseInput(id=id, title=request.title, description=request.description)
            )
        except RepositoryNotFoundError:
            raise NotFoundError(message="Todo not found")
        return UpdateTodoResponse(
            id=str(result.todo.id),
            title=result.todo.title,
            description=result.todo.description,
            completed=result.todo.completed,
        )
```

`main.py`ではComposition
Rootとしてすべての依存を組み立て、ミドルウェアを登録します。この構造はNode.js編の`main.ts`、Go編の`cmd/server/main.go`と対応しています。

```python
# backend_python/main.py（抜粋）
engine = create_engine(DATABASE_URL, echo=True)
context_provider = ContextProviderImpl(engine=engine)
todo_repository = TodoRepositoryImpl(context_provider=context_provider)
transaction_service = TransactionServiceImpl(context_provider=context_provider)

create_todo_usecase = CreateTodoUsecaseImpl(todo_repository=todo_repository)
update_todo_usecase = UpdateTodoUsecaseImpl(
    todo_repository=todo_repository, transaction_service=transaction_service
)

app.add_middleware(SessionMiddleware, context_provider=context_provider)
app.add_middleware(ErrorHandler)
app.include_router(todo_router(todo_router_container))
```

## テスト戦略

Python版もレイヤ単位でテストを配置しています。

- domain: エンティティの状態遷移とDTO変換を確認（`tests/domain/test_todo.py`）
- application_service: ユースケースの入出力と例外ハンドリングを確認
- infrastructure: コンテキストプロバイダやトランザクションサービスがコミット/ロールバックすることを実 DB で検証
- presentation: ミドルウェアとハンドラのエラーマッピング、ルータのHTTP仕様をテスト

特に`tests/infrastructure/service/test_transaction_service.py`では、ネストされたトランザクションが外側に影響しないことを確認しています。SQLAlchemyの制限に対して仕組みが正しく機能するかを自動テストで担保しているのがポイントです。

また、インフラ層では`session.merge()`と`ON DUPLICATE KEY`に相当するMySQL固有の挙動を利用してアップサートを成立させているため、SQLiteなどの簡易DBに切り替えると動作差分が出てしまいます。この理由から`tests/infrastructure/conftest.py`ではTestcontainersでMySQLを起動し、実際のSQL方言と同じ環境で検証しています。

## まとめ

- SQLAlchemyのセッションは`ContextVar`とミドルウェアでリクエスト境界に閉じ込め、ネスト時は`begin_nested()`で吸収
- ドメインはORMを知らず、DTO経由で永続化層とやり取りすることでテスタビリティと整合性を確保
- トランザクションは`TransactionService`を介してユースケースに集中させ、FastAPIの例外ハンドリングと連携してHTTPレスポンスを統一

PythonでもDDD＋クリーンアーキテクチャの骨格はそのままに、言語固有の制約をインフラレイヤで解消すれば快適に進められます。

次回はJavaでの実装例を紹介します。お楽しみに！
