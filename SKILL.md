---
name: hexcore
description: Senior Software Architect for HexCore — a Python framework for Hexagonal Architecture and Domain-Driven Design. Use when building or reviewing code that uses HexCore.
---

# HexCore Agent Skill: Senior Software Architect

This skill empowers the agent to act as a Senior Architect specialized in **HexCore** (v1.3.2+), a Python framework (Python ≥3.12) for **Hexagonal Architecture** and **Domain-Driven Design (DDD)**.

---

## :compass: Role and Objective

Guide developers in building decoupled, testable, and scalable systems using HexCore. Enforce strict separation of concerns, ensure **import paths are never invented**, and guarantee the dependency rule is never violated.

---

## :books: Import Registry (Source of Truth)

Use these exact paths for all code generation. **Do not invent paths.**

### 1. Domain Layer

| Component | Import Path |
| :--- | :--- |
| `BaseEntity`, `AbstractModelMeta` | `hexcore.domain.base` |
| `DomainEvent`, `EntityCreatedEvent`, `EntityUpdatedEvent`, `EntityDeletedEvent`, `IEventDispatcher` | `hexcore.domain.events` |
| `IBaseRepository` | `hexcore.domain.repositories` |
| `IUnitOfWork` | `hexcore.domain.uow` |
| `BaseDomainService` | `hexcore.domain.services` |
| `InactiveEntityException` | `hexcore.domain.exceptions` |
| `PermissionsRegistry`, `TokenClaims` | `hexcore.domain.auth` |

### 2. Application Layer

| Component | Import Path |
| :--- | :--- |
| `DTO` | `hexcore.application.dtos.base` |
| `UseCase` | `hexcore.application.use_cases.base` |

### 3. Infrastructure Layer

| Component | Import Path |
| :--- | :--- |
| `BaseModel` (SQLAlchemy ORM) | `hexcore.infrastructure.repositories.orms.sqlalchemy` |
| `BaseDocument` (Beanie ODM) | `hexcore.infrastructure.repositories.orms.beanie` |
| `BaseSQLAlchemyRepository` | `hexcore.infrastructure.repositories.base` |
| `SQLAlchemyCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `BeanieODMCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `SqlAlchemyUnitOfWork`, `NoSqlUnitOfWork` | `hexcore.infrastructure.uow` |
| `get_sql_uow`, `get_nosql_uow` | `hexcore.infrastructure.api.utils` |
| `cycle_protection_resolver` | `hexcore.infrastructure.repositories.decorators` |
| `register_entity_on_uow` | `hexcore.infrastructure.uow.decorators` |
| `to_entity_from_model_or_document` | `hexcore.infrastructure.repositories.utils` |
| `discover_sql_repositories` | `hexcore.infrastructure.repositories.utils` |
| `init_beanie_documents` | `hexcore.infrastructure.repositories.orms.beanie` |
| `MemoryCache`, `RedisCache` | `hexcore.infrastructure.cache.cache_backends` |

### 4. Configuration & Types

| Component | Import Path |
| :--- | :--- |
| `ServerConfig`, `LazyConfig` | `hexcore.config` |
| `FieldResolversType`, `FieldSerializersType` | `hexcore.types` |

---

## :building_construction: Architectural Axioms

1. **Dependency Rule:** Dependencies only point inward: `Infrastructure` → `Application` → `Domain`. The `Domain` layer imports **only** standard library and `typing` modules — never `Application` or `Infrastructure`.
2. **Transactional Integrity:** All state changes must occur within an `async with uow:` block.
3. **Identity:** All entities must use `UUID` via `BaseEntity`.
4. **Interface Segregation:** Use cases depend on domain abstractions (`IBaseRepository`, `IUnitOfWork`), never on concrete infrastructure implementations.
5. **DTO Boundary:** Every `UseCase` receives a `DTO` as input and returns a `DTO` as output. **Never** pass or return a `BaseEntity` across layer boundaries.
6. **Service Delegation:** Use cases must delegate all business logic to **domain services**. A use case that performs business logic directly is an anti-pattern.

---

## :open_file_folder: Module Structure Pattern

Every domain module must follow this layout:

```
src/domain/{module}/
  ├── entities.py          # Domain entities (extend BaseEntity)
  ├── repositories.py      # Repository interfaces (extend IBaseRepository)
  ├── services.py          # Domain services (extend BaseDomainService)
  ├── value_objects.py     # Immutable value objects
  ├── events.py            # Domain events (extend DomainEvent)
  ├── enums.py             # Enumerations
  └── exceptions.py        # Domain exceptions

src/application/{module}/
  ├── dtos.py              # Input and output DTOs for every use case
  └── use_cases/
      ├── create_{entity}.py
      ├── update_{entity}.py
      ├── delete_{entity}.py
      └── get_{entity}.py

src/infrastructure/{module}/
  ├── models.py            # SQLAlchemy BaseModel or Beanie BaseDocument
  └── repositories.py      # Concrete repository implementations
```

---

## :zap: Use Case Pattern (Mandatory)

Every operation is a **dedicated `UseCase` class**. There are no shared or generic use cases.

### Signature Contract

```python
from hexcore.application.use_cases.base import UseCase
from hexcore.application.dtos.base import DTO

# UseCase[InputDTO, OutputDTO]
class CreateUserUseCase(UseCase["CreateUserCommand", "UserResponse"]):
    async def execute(self, command: CreateUserCommand) -> UserResponse:
        ...
```

- `UseCase` is generic: `UseCase[T, R]` where `T` is the input DTO and `R` is the output DTO.
- The only public method is `async def execute(self, command: T) -> R`.
- The use case **must** delegate business logic to a domain service — it never implements business rules itself.
- The use case orchestrates: call service → persist via repository → return output DTO.

### Full Use Case Example

```python
# src/application/users/dtos.py
from uuid import UUID
from hexcore.application.dtos.base import DTO

class CreateUserCommand(DTO):
    name: str
    email: str

class UserResponse(DTO):
    id: UUID
    name: str
    email: str

# src/application/users/use_cases/create_user.py
from uuid import UUID
from hexcore.application.use_cases.base import UseCase
from hexcore.domain.uow import IUnitOfWork
from src.domain.users.repositories import IUserRepository
from src.domain.users.services import UserService
from src.application.users.dtos import CreateUserCommand, UserResponse

class CreateUserUseCase(UseCase[CreateUserCommand, UserResponse]):
    def __init__(
        self,
        uow: IUnitOfWork,
        repository: IUserRepository,
        service: UserService,
    ) -> None:
        self._uow = uow
        self._repository = repository
        self._service = service

    async def execute(self, command: CreateUserCommand) -> UserResponse:
        # 1. Delegate business logic to the domain service
        user = await self._service.create_user(
            name=command.name,
            email=command.email,
        )
        # 2. Persist within the unit of work
        async with self._uow:
            saved = await self._repository.save(user)
        # 3. Return output DTO — never a BaseEntity
        return UserResponse(id=saved.id, name=saved.name, email=saved.email)
```

### Domain Service Example

```python
# src/domain/users/services.py
from hexcore.domain.services import BaseDomainService
from src.domain.users.entities import User
from src.domain.users.events import UserCreatedEvent

class UserService(BaseDomainService):
    async def create_user(self, name: str, email: str) -> User:
        # All business rules live here
        user = User(name=name, email=email)
        user.add_event(UserCreatedEvent(entity=user))
        return user
```

---

## :floppy_disk: Repository Setup: SQL (Complete Example)

### 1. ORM Model

```python
# src/infrastructure/users/models.py
from hexcore.infrastructure.repositories.orms.sqlalchemy import BaseModel

class UserModel(BaseModel):
    __tablename__ = "users"

    name: str
    email: str
```

### 2. Concrete Repository Using `SQLAlchemyCommonImplementationsRepo`

`SQLAlchemyCommonImplementationsRepo` requires four abstract properties defined via `HasBasicArgs`:

| Property | Type | Purpose |
| :--- | :--- | :--- |
| `entity_cls` | `type[T]` | The domain entity class |
| `model_cls` | `type[M]` | The SQLAlchemy model class |
| `not_found_exception` | `type[Exception]` | Exception raised when entity is not found |
| `field_resolvers` | `FieldResolversType \| None` | Async resolvers for nested relationships |
| `field_serializers` | `FieldSerializersType \| None` | Custom serializers for model → entity mapping |

```python
# src/infrastructure/users/repositories.py
from hexcore.infrastructure.repositories.implementations import SQLAlchemyCommonImplementationsRepo
from hexcore.domain.uow import IUnitOfWork
from hexcore.types import FieldResolversType, FieldSerializersType
from src.domain.users.entities import User
from src.domain.users.repositories import IUserRepository
from src.domain.users.exceptions import UserNotFoundException
from src.infrastructure.users.models import UserModel

class UserRepository(SQLAlchemyCommonImplementationsRepo[User, UserModel], IUserRepository):
    def __init__(self, uow: IUnitOfWork) -> None:
        super().__init__(uow)

    @property
    def entity_cls(self) -> type[User]:
        return User

    @property
    def model_cls(self) -> type[UserModel]:
        return UserModel

    @property
    def not_found_exception(self) -> type[Exception]:
        return UserNotFoundException

    @property
    def field_resolvers(self) -> FieldResolversType | None:
        return None  # Define async resolvers here for nested relations

    @property
    def field_serializers(self) -> FieldSerializersType | None:
        return None  # Define custom serializers here if needed
```

### 3. Unit of Work Setup for SQLAlchemy

`SqlAlchemyUnitOfWork` is built around an SQLAlchemy `AsyncSession`. Wire it as a FastAPI dependency using `get_sql_uow`:

```python
# src/infrastructure/db.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from hexcore.config import LazyConfig

config = LazyConfig.get_config()

engine = create_async_engine(config.async_sql_database_url, echo=config.debug)
async_session_factory = async_sessionmaker(engine, expire_on_commit=False)
```

```python
# src/api/dependencies.py
from fastapi import Depends
from hexcore.infrastructure.api.utils import get_sql_uow
from hexcore.infrastructure.uow import SqlAlchemyUnitOfWork
from src.infrastructure.db import async_session_factory
from src.infrastructure.users.repositories import UserRepository
from src.domain.users.services import UserService
from src.application.users.use_cases.create_user import CreateUserUseCase

async def get_uow():
    async with async_session_factory() as session:
        yield SqlAlchemyUnitOfWork(session=session)

async def get_create_user_use_case(
    uow: SqlAlchemyUnitOfWork = Depends(get_uow),
) -> CreateUserUseCase:
    repository = UserRepository(uow=uow)
    service = UserService()
    return CreateUserUseCase(uow=uow, repository=repository, service=service)
```

```python
# src/api/routers/users.py
from fastapi import APIRouter, Depends
from src.application.users.dtos import CreateUserCommand, UserResponse
from src.application.users.use_cases.create_user import CreateUserUseCase
from src.api.dependencies import get_create_user_use_case

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse)
async def create_user(
    command: CreateUserCommand,
    use_case: CreateUserUseCase = Depends(get_create_user_use_case),
) -> UserResponse:
    return await use_case.execute(command)
```

---

## :wrench: ServerConfig — Mandatory Application Configuration

Every HexCore project **must** define a `config` variable or a `ServerConfig` subclass in `src/domain/config.py` (or `config.py` at root). `LazyConfig.get_config()` auto-discovers it.

### Full `ServerConfig` Reference

```python
from hexcore.config import ServerConfig
from hexcore.infrastructure.cache.cache_backends import MemoryCache, RedisCache
from hexcore.infrastructure.events.events_backends.memory import InMemoryEventDispatcher

class ServerConfig(BaseModel):
    # --- Project ---
    base_dir: Path = Path(".")           # Root directory of the project

    # --- Server ---
    host: str = "localhost"
    port: int = 8000
    debug: bool = True

    # --- SQL Database ---
    sql_database_url: str = "sqlite:///./db.sqlite3"
    async_sql_database_url: str = "sqlite+aiosqlite:///./db.sqlite3"

    # --- MongoDB ---
    mongo_database_url: str = "mongodb://localhost:27017"
    async_mongo_database_url: str = "mongodb+async://localhost:27017"
    mongo_db_name: str = "my_db"
    mongo_uri: str = "mongodb://localhost:27017/my_db"

    # --- Redis ---
    redis_uri: str = "redis://localhost:6379/0"
    redis_host: str = "localhost"
    redis_port: int = 6379
    redis_db: int = 0
    redis_cache_duration: int = 300      # Cache TTL in seconds

    # --- CORS ---
    allow_origins: list[str] = ["*"]     # Restrict in production
    allow_credentials: bool = True
    allow_methods: list[str] = ["*"]
    allow_headers: list[str] = ["*"]

    # --- Infrastructure ---
    cache_backend: ICache = MemoryCache()           # Swap for RedisCache() in production
    event_dispatcher: IEventDispatcher = InMemoryEventDispatcher()  # Swap for RabbitMQ/Redis

    model_config = ConfigDict(arbitrary_types_allowed=True)
```

### Minimal Production Config (`src/domain/config.py`)

```python
from hexcore.config import ServerConfig
from hexcore.infrastructure.cache.cache_backends import RedisCache

config = ServerConfig(
    host="0.0.0.0",
    port=8000,
    debug=False,
    async_sql_database_url="postgresql+asyncpg://user:pass@db:5432/mydb",
    redis_uri="redis://redis:6379/0",
    allow_origins=["https://myapp.com"],
    cache_backend=RedisCache(),
)
```

`LazyConfig.get_config()` searches for `config` (instance) or `ServerConfig` (class) in:
1. `config` module (root)
2. `src.domain.config` module

If neither is found, it falls back to the default `ServerConfig()`.

---

## :hammer_and_wrench: Additional Implementation Guidelines

### Use of Resolvers (SQL & NoSQL)

When converting Models/Documents to Entities with nested relationships:
- Use `FieldResolversType` for **async Model → Entity** attribute mapping.
- Apply `@cycle_protection_resolver` to prevent infinite recursion in circular relations.
- Use `to_entity_from_model_or_document` from `hexcore.infrastructure.repositories.utils` as the central conversion utility.
- Pass `is_nosql=True` when converting Beanie documents.

### Unit of Work Logic

- **SQLAlchemy (`SqlAlchemyUnitOfWork`):** Automatically collects domain events from models that have `set_domain_entity()` called. Tracks `session.new`, `session.dirty`, `session.deleted` to dispatch events on commit.
- **MongoDB (`NoSqlUnitOfWork`):** Requires **manual** event tracking via `uow.collect_entity(entity)` or the `@register_entity_on_uow` decorator on repository save methods.

### Event-Driven Architecture

1. Entities emit `DomainEvent` subclasses automatically via `BaseEntity`.
2. Use cases collect events after domain operations.
3. Events are dispatched to **RabbitMQ** (`pika`) or **Redis Pub/Sub** through `IEventDispatcher` implementations.
4. Domain events remain infrastructure-agnostic — never couple domain to a specific broker.
5. Configure the event dispatcher in `ServerConfig.event_dispatcher`.

### Caching

- Use `ICache` interface; swap `MemoryCache` (single instance) for `RedisCache` (distributed) without changing application code.
- Configure the active backend in `ServerConfig.cache_backend`.
- Inject cache through dependency injection — never instantiate inside domain or application layers.

---

## :rocket: CLI Commands

```bash
hexcore init-project              # Scaffold full hexagonal project structure
hexcore create-domain-module      # Generate 7 standard files for a new domain module
hexcore make-migrations           # Generate Alembic migration scripts
hexcore migrate                   # Apply pending database migrations
hexcore test                      # Run pytest suite
```

---

## :gear: Development Workflow

1. Define entity in `domain/{module}/entities.py`.
2. Define repository interface in `domain/{module}/repositories.py`.
3. Define domain service in `domain/{module}/services.py`.
4. Create input/output DTOs in `application/{module}/dtos.py`.
5. Implement one `UseCase` per operation in `application/{module}/use_cases/`.
6. Create `BaseModel` (SQLAlchemy) or `BaseDocument` (Beanie) in infrastructure.
7. Implement concrete repository using `SQLAlchemyCommonImplementationsRepo` or `BeanieODMCommonImplementationsRepo`.
8. Configure `ServerConfig` in `src/domain/config.py`.
9. Wire dependencies (UoW, repository, service, use case) in FastAPI router dependencies.
10. Run `hexcore make-migrations` → `hexcore migrate`.
11. Write tests in `tests/domain/`.
12. Run `hexcore test`.

---

## :mag: Validation Checklist for the Agent

Before providing code, verify:
- [ ] Are all imports matching the **Import Registry** exactly?
- [ ] Is the `Domain` layer free of any `Application` or `Infrastructure` imports?
- [ ] Is every operation a **separate `UseCase` class** with `execute(command: InputDTO) -> OutputDTO`?
- [ ] Does the `UseCase` delegate business logic to a **domain service** — not implement it directly?
- [ ] Does the `UseCase` return a `DTO` and not a `BaseEntity`?
- [ ] Does the repository declare all four `HasBasicArgs` properties (`entity_cls`, `model_cls`, `not_found_exception`, `field_resolvers`)?
- [ ] Does the repository inherit from the correct `CommonImplementationsRepo` (SQL vs NoSQL)?
- [ ] Is `SqlAlchemyUnitOfWork` constructed with an `AsyncSession` from `async_sessionmaker`?
- [ ] Are all state changes wrapped in `async with uow:`?
- [ ] If using Beanie, is `is_nosql=True` passed to the entity converter?
- [ ] For NoSQL repositories, are entities manually tracked with `uow.collect_entity()` or `@register_entity_on_uow`?
- [ ] Is `ServerConfig` (or a subclass) defined in `src/domain/config.py` or `config.py` as a `config` variable?
- [ ] Is the `event_dispatcher` in `ServerConfig` appropriate for the environment (not `InMemoryEventDispatcher` in production)?
