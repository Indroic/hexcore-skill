---
name: hexcore
description: A brief description of what this skill does
---

# hexcore

---

## 🧭 What is HexCore?

HexCore is a reusable base module for Python projects implementing **Hexagonal Architecture** and **Domain-Driven Design (DDD)**. It provides base classes, abstractions, contracts, and utilities for entities, repositories, services, events, cache, and Unit of Work (UoW). [1](#0-0) 

- **Install:** `pip install hexcore`
- **Python:** >= 3.12
- **Key deps:** Pydantic v2, SQLAlchemy, Beanie, FastAPI, Redis, Alembic, Typer [2](#0-1) 

---

## 🗂️ Project Layer Structure

A project using HexCore should follow this layout:

```
src/
├── domain/
│   └── {module}/
│       ├── entities.py
│       ├── repositories.py
│       ├── services.py
│       ├── value_objects.py
│       ├── events.py
│       ├── enums.py
│       └── exceptions.py
├── application/
│   ├── dtos/
│   └── use_cases/
└── infrastructure/
    └── database/
        ├── models/        ← SQLAlchemy models
        └── documents/     ← Beanie documents
``` [3](#0-2) 

---

## ⚙️ Configuration

### `ServerConfig`

The central configuration Pydantic model. Extend it to override defaults: [4](#0-3) 

Key fields:

| Field | Default |
|---|---|
| `sql_database_url` | `sqlite:///./db.sqlite3` |
| `async_sql_database_url` | `sqlite+aiosqlite:///./db.sqlite3` |
| `mongo_uri` | `mongodb://localhost:27017/euphoria_db` |
| `redis_uri` | `redis://localhost:6379/0` |
| `cache_backend` | `MemoryCache()` |
| `event_dispatcher` | `InMemoryEventDispatcher()` |

### `LazyConfig`

Loads the project config lazily. Searches for a `config` variable or `ServerConfig` class in `config` or `src.domain.config` modules. Falls back to defaults. [5](#0-4) 

**Usage pattern:**
```python
from hexcore.config import LazyConfig
config = LazyConfig.get_config()
```

Place your custom config in `config.py` at the project root OR in `src/domain/config.py`:
```python
# config.py
from hexcore.config import ServerConfig
config = ServerConfig(
    async_sql_database_url="postgresql+asyncpg://user:pass@localhost/mydb",
    mongo_uri="mongodb://localhost:27017/mydb",
)
```

---

## 🏛️ Domain Layer

### `BaseEntity`

Base Pydantic class for all domain entities. Provides common fields and domain event management. [6](#0-5) 

**Auto-provided fields:**
- `id: UUID` — auto-generated with `uuid4()`
- `created_at: datetime` — UTC timestamp on creation
- `updated_at: datetime` — UTC timestamp, update manually
- `is_active: bool` — soft-delete flag (default `True`)

**Domain event methods:**
- `register_event(event)` — queue a domain event on the entity
- `pull_domain_events()` — returns all queued events and clears the list
- `clear_domain_events()` — clears without returning
- `async deactivate()` — sets `is_active = False`

**Config rules (`model_config`):**
- `from_attributes=True` — ORM-friendly (works with SQLAlchemy)
- `validate_assignment=True` — re-validates on field assignment
- `frozen=False` — entities are mutable

### `AbstractModelMeta`

Mix-in that resolves the metaclass conflict between Pydantic and `abc.ABC`. Use this as a base when you need `@abstractmethod` on a Pydantic entity. [7](#0-6) 

---

### Domain Events [8](#0-7) 

#### `DomainEvent` — base class for all events
- `event_id: UUID` — auto-generated
- `occurred_on: datetime` — UTC timestamp
- `event_name: str` (computed) — class name without "Event", uppercased
- Immutable (`frozen=True`)

#### Built-in event types:

| Class | Purpose | Extra fields |
|---|---|---|
| `EntityCreatedEvent[T]` | Entity created | `entity_id`, `entity_data: T` |
| `EntityUpdatedEvent[T]` | Entity updated | `entity_id`, `entity_data: T` |
| `EntityDeletedEvent` | Entity deleted | `entity_id` |

#### `IEventDispatcher`

Port (interface) for event dispatching:
- `register(event_type, handler)` — register a handler for an event type
- `async dispatch(event)` — dispatch an event

#### `EventHandler` type alias
```python
EventHandler = Callable[[DomainEvent], Awaitable[None]]
```

---

### `IBaseRepository[T]`

Generic async CRUD interface all repositories must implement: [9](#0-8) 

- `async get_by_id(entity_id: UUID) -> T`
- `async get_active_by_id(entity_id: UUID) -> T` — raises `InactiveEntityException` if `is_active=False`
- `async list_all() -> List[T]`
- `async save(entity: T) -> T`
- `async delete(entity: T) -> None`

---

### `IUnitOfWork`

Transactional context manager interface: [10](#0-9) 

- Use as `async with uow:` — automatically calls `rollback()` on exit
- `async commit()` — persist and dispatch events
- `async rollback()` — undo pending changes
- `collect_domain_events()` — gather all queued events
- `async dispatch_events()` — fire all collected events
- `collect_entity(entity)` — manually track an entity for event dispatch
- `clear_tracked_entities()` — remove all tracked entities

---

### `BaseDomainService`

Base class for domain services. Auto-injects `config` and `event_dispatcher` from `LazyConfig`: [11](#0-10) 

---

### `InactiveEntityException`

Raised by `get_active_by_id` when the entity has `is_active=False`: [12](#0-11) 

---

### Auth Module

#### `PermissionsRegistry`

Dynamic permission registration and lookup. Does not define concrete permissions — each project registers its own: [13](#0-12) 

Methods:
- `register_permission(name, value=None)` — registers one permission
- `register_permissions(dict)` — batch register
- `get_all_permission_values()` — returns `Set[str]` of all values
- `get_permission_by_name(name)` — lookup by name
- `build_permissions_enum()` — creates a dynamic `Enum`

#### `TokenClaims`

Value object for JWT/OAuth token claims: [14](#0-13) 

Fields: `iss`, `sub`, `exp`, `iat`, `jti` (auto-uuid), `client_id`, `scopes`.

---

## 📦 Application Layer

### `DTO`

Base class for all Data Transfer Objects. Inherits Pydantic `BaseModel`: [15](#0-14) 

### `UseCase[T, R]`

Generic async use case with a single `execute` method. `T` = input DTO, `R` = output DTO: [16](#0-15) 

Pattern:
```python
class CreateUserCommand(DTO):
    name: str

class CreateUserResult(DTO):
    id: UUID

class CreateUserUseCase(UseCase[CreateUserCommand, CreateUserResult]):
    async def execute(self, command: CreateUserCommand) -> CreateUserResult:
        ...
```

---

## 🏗️ Infrastructure Layer

### SQLAlchemy ORM

#### `BaseModel[T]` (SQLAlchemy)

Abstract SQLAlchemy declarative base with common mapped columns. Extend for every SQL table: [17](#0-16) 

Fields auto-mapped: `id (UUID PK)`, `is_active (Boolean)`, `created_at (DateTime TZ)`, `updated_at (DateTime TZ, auto-updates)`.

Methods:
- `set_domain_entity(entity)` / `get_domain_entity()` — link ORM model to domain entity

#### `Base` (DeclarativeBase)

Import `Base.metadata` in Alembic's `env.py` for auto-migrations.

#### `AsyncSessionLocal`

Pre-configured async session factory: [18](#0-17) 

#### `get_async_db_session()`

Dependency generator for a raw async DB session: [19](#0-18) 

---

### Beanie ODM

#### `BaseDocument`

Abstract Beanie document base. Note: `entity_id` maps to the domain entity's `id`: [20](#0-19) 

Fields: `entity_id (UUID, unique indexed)`, `created_at`, `updated_at` (auto-updates on save), `is_active`.

Settings: `is_root=True`, `use_cache=True`.

---

### `BaseSQLAlchemyRepository[T]`

Intermediate base that wires the `AsyncSession` from the UoW: [21](#0-20) 

Access the session via `self.session`.

---

### `HasBasicArgs[T, A]` — Repository Property Protocol

Mixin that defines required properties for concrete repo implementations: [22](#0-21) 

Override these properties:
- `entity_cls` — your domain entity class
- `not_found_exception` — exception class raised when entity not found
- `fields_resolvers` — `FieldResolversType` for Model→Entity conversions (complex fields)
- `fields_serializers` — `FieldSerializersType` for Entity→Model conversions (complex fields)

---

### `SQLAlchemyCommonImplementationsRepo[T, M]`

Full CRUD repo for SQL. Inherits `BaseSQLAlchemyRepository` + `HasBasicArgs`. Override `model_cls` property: [23](#0-22) 

Additional property to override: `model_cls` — your SQLAlchemy `BaseModel` subclass.

---

### `BeanieODMCommonImplementationsRepo[T, D]`

Full CRUD repo for MongoDB/Beanie. Inherits `IBaseRepository` + `HasBasicArgs`. Override `document_cls` property: [24](#0-23) 

Additional property to override: `document_cls` — your `BaseDocument` subclass.

The `save()` method is decorated with `@register_entity_on_uow`, so entities with pending events are auto-tracked.

---

### Unit of Work Implementations

#### `SqlAlchemyUnitOfWork` [25](#0-24) 

- Takes `AsyncSession` in constructor
- `commit()` → flushes to DB + dispatches events
- `rollback()` → rolls back session + clears tracked entities
- Auto-collects domain events from session-tracked SQLAlchemy models via `get_domain_entity()`
- `__aexit__` closes the session

#### `NoSqlUnitOfWork` [26](#0-25) 

- No-arg constructor
- Manually tracks entities via `collect_entity(entity)`
- `rollback()` clears domain events on all tracked entities

---

### FastAPI Dependency Helpers [27](#0-26) 

- `get_session()` — yields `AsyncSession`
- `get_sql_uow(session)` — yields `SqlAlchemyUnitOfWork` (use with `Depends`)
- `get_nosql_uow()` — yields `NoSqlUnitOfWork` (use with `Depends`)

---

### Cache Backends

#### `ICache` (interface) [28](#0-27) 

Methods: `async get(key)`, `set(key, value, expire)`, `delete(key)`.

#### `MemoryCache`

In-memory dict cache. Default backend. No expiry support: [29](#0-28) 

#### `RedisCache`

Redis-backed cache via `redis.asyncio`. Reads `redis_uri` and `redis_cache_duration` from `LazyConfig`: [30](#0-29) 

---

### Event Dispatcher Backends

#### `InMemoryEventDispatcher`

Default dispatcher. Stores events as a list of `(class_name, dict)` tuples. Does not actually call handlers — for testing/dev use: [31](#0-30) 

For production, implement `IEventDispatcher` with a real broker (RabbitMQ, Redis Pub/Sub, etc.).

---

## 🔧 Types Reference (`hexcore/types.py`) [32](#0-31) 

| Type Alias | Shape | Purpose |
|---|---|---|
| `FieldResolversType[A]` | `Dict[str, Tuple[str, AsyncCycleResolver[A]]]` | Model→Entity async field resolvers |
| `FieldSerializersType[A]` | `Dict[str, Tuple[str, Callable[[A], Any]]]` | Entity→Model sync field serializers |
| `RelationsType` | `Dict[str, Tuple[Type[BaseModel], List[UUID]]]` | SQLAlchemy relation assignment |
| `ExcludeType` | `Optional[set[str]]` | Fields to exclude during dump |
| `AsyncCycleResolver[A]` | `Callable[[A], Awaitable[Any]]` | Single-arg async resolver |

**`FieldResolversType` format:**
```python
{
    "entity_field_name": ("model_field_name", async_resolver_function),
}
```
**`FieldSerializersType` format:**
```python
{
    "entity_field_name": ("model_field_name", sync_serializer_function),
}
```

---

## 🛡️ Decorators

### `@cycle_protection_resolver`

Wraps an async resolver to prevent infinite recursion in cyclic entity relationships. Uses `contextvars` for safe async context isolation: [33](#0-32) 

Use on any field resolver function that may resolve back to the same entity type.

### `@register_entity_on_uow`

Wraps a repo `save()` to auto-register entities with pending domain events into the UoW for dispatch: [34](#0-33) 

---

## 🔄 Utility Functions

### `to_entity_from_model_or_document`

Converts a SQLAlchemy model or Beanie document instance into a domain entity, applying async field resolvers: [35](#0-34) 

- Set `is_nosql=True` for Beanie documents (renames `entity_id` → `id`)
- `field_resolvers` is a `FieldResolversType` dict

### SQLAlchemy Utilities (`hexcore.infrastructure.repositories.orms.sqlalchemy.utils`) [36](#0-35) 

| Function | Purpose |
|---|---|
| `to_model(entity, model_cls, ...)` | Domain entity → SQLAlchemy model |
| `db_get(session, model, id, exc_none)` | Fetch by PK, raise exc if not found |
| `db_list(session, model)` | List all with eager-loaded relations |
| `db_save(session, entity)` | Merge + flush + refresh |
| `save_entity(session, entity, model_cls, ...)` | Full save pipeline with serializers and relations |
| `logical_delete(session, entity, model_cls)` | Soft-delete via `deactivate()` |
| `assign_relations(session, model, relations)` | Populate SQLAlchemy relationship fields from UUIDs |
| `load_relations(model)` | Auto-detect and `selectinload` all relations |
| `select_in_load_options(*rels, model)` | Selective `selectinload` for named relations |
| `import_all_models(package)` | Import all submodules (triggers SQLAlchemy registration) |

### Beanie Utilities (`hexcore.infrastructure.repositories.orms.beanie.utils`) [37](#0-36) 

| Function | Purpose |
|---|---|
| `to_document(entity, document_cls, ...)` | Domain entity → Beanie document |
| `discover_beanie_documents()` | Finds all concrete `BaseDocument` subclasses |
| `async init_beanie_documents()` | Connects to MongoDB and initializes all documents |
| `async db_get(document_cls, entity_id)` | Find document by `entity_id` |
| `async db_list(document_cls)` | List all documents |
| `async save_entity(entity, document_cls, ...)` | Upsert document |
| `async logical_delete(entity_id, document_cls)` | Set `is_active=False` |

### Repository Discovery [38](#0-37) 

`discover_sql_repositories()` — returns `Dict[str, Type[BaseSQLAlchemyRepository]]` mapping lowercase repo names to their classes (e.g., `"user"` → `UserRepository`).

### UoW Helper [39](#0-38) 

`get_repository(uow, repo_name, repo_type)` — type-safe accessor for a named repo from a UoW instance.

---

## 💻 CLI Commands

The CLI is exposed as `hexcore` (entry point) or via `manage.py` generated by `init`: [40](#0-39) 

| Command | Args | Purpose |
|---|---|---|
| `hexcore init <project_name>` | project name | Scaffolds full project structure + Alembic |
| `hexcore create-domain-module <name>` | module name | Creates domain module files + test folder |
| `hexcore make-migrations <description>` | description | `alembic revision --autogenerate` |
| `hexcore migrate` | — | `alembic upgrade head` |
| `hexcore test [path]` | optional path | Runs pytest |

The `create-domain-module` command generates:
- `repositories.py` with `I{Name}Repository(IBaseRepository[{Name}])`
- `services.py` with `{Name}Service(BaseDomainService)`
- `value_objects.py` with commented template
- `events.py` with `EntityCreatedEvent` template
- `enums.py`, `exceptions.py` (blank) [41](#0-40) 

---

## 📐 Architecture Flow Diagram

```mermaid
flowchart TD
    "FastAPI Route" --> "UseCase.execute(DTO)"
    "UseCase.execute(DTO)" --> "DomainService / IBaseRepository"
    "DomainService / IBaseRepository" --> "IUnitOfWork"
    "IUnitOfWork" --> "SQLAlchemy AsyncSession / Beanie"
    "SQLAlchemy AsyncSession / Beanie" --> "BaseModel / BaseDocument"
    "BaseModel / BaseDocument" --> "to_entity_from_model_or_document"
    "to_entity_from_model_or_document" --> "BaseEntity"
    "BaseEntity" --> "register_event(DomainEvent)"
    "IUnitOfWork" --> "commit() → dispatch_events()"
    "commit() → dispatch_events()" --> "IEventDispatcher"
    "IEventDispatcher" --> "EventHandler"
```

---

## 📌 Key Import Paths Cheat Sheet

| What | Import path |
|---|---|
| `BaseEntity` | `hexcore.domain.base` |
| `DomainEvent`, `EntityCreatedEvent`, etc. | `hexcore.domain.events` |
| `IBaseRepository` | `hexcore.domain.repositories` |
| `IUnitOfWork` | `hexcore.domain.uow` |
| `BaseDomainService` | `hexcore.domain.services` |
| `InactiveEntityException` | `hexcore.domain.exceptions` |
| `PermissionsRegistry` | `hexcore.domain.auth.permissions` |
| `TokenClaims` | `hexcore.domain.auth.value_objects` |
| `DTO` | `hexcore.application.dtos.base` |
| `UseCase` | `hexcore.application.use_cases.base` |
| `BaseModel` (SQLAlchemy) | `hexcore.infrastructure.repositories.orms.sqlalchemy` |
| `BaseDocument` (Beanie) | `hexcore.infrastructure.repositories.orms.beanie` |
| `BaseSQLAlchemyRepository` | `hexcore.infrastructure.repositories.base` |
| `SQLAlchemyCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `BeanieODMCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `SqlAlchemyUnitOfWork`, `NoSqlUnitOfWork` | `hexcore.infrastructure.uow` |
| `ICache` | `hexcore.infrastructure.cache` |
| `MemoryCache` | `hexcore.infrastructure.cache.cache_backends.memory` |
| `RedisCache` | `hexcore.infrastructure.cache.cache_backends.redis` |
| `InMemoryEventDispatcher` | `hexcore.infrastructure.events.events_backends.memory` |
| `get_sql_uow`, `get_nosql_uow` | `hexcore.infrastructure.api.utils` |
| `AsyncSessionLocal` | `hexcore.infrastructure.repositories.orms.sqlalchemy.session` |
| `ServerConfig`, `LazyConfig` | `hexcore.config` |
| `FieldResolversType`, `FieldSerializersType` | `hexcore.types` |
| `cycle_protection_resolver` | `hexcore.infrastructure.repositories.decorators` |
| `init_beanie_documents` | `hexcore.infrastructure.repositories.orms.beanie.utils` | [42](#0-41) 

---

## Notes

- **`entity_id` vs `id`:** Beanie documents use `entity_id` internally (to avoid conflict with MongoDB's `_id`), but `to_entity_from_model_or_document` with `is_nosql=True` automatically renames it back to `id` for the domain entity. [43](#0-42) 
- **Events are NOT dispatched automatically in SQL:** The UoW collects events from tracked SQLAlchemy models only when `commit()` is called. The model must have `set_domain_entity()` called during `save_entity()` for this to work. [44](#0-43) 
- **`InMemoryEventDispatcher` does not call handlers:** It only stores events as a list. For production, implement a custom `IEventDispatcher`. [45](#0-44) 
- **`FieldResolversType` key is `(data_field, resolver)`:** The first element of the tuple is the source field name on the model dict, the second is the async function receiving the full model instance. [46](#0-45) 
- **`LazyConfig` is a singleton:** The first call to `get_config()` caches the result in `_imported_config`; subsequent calls return the cached instance. [47](#0-46) 
- **Alembic integration:** Use `import_all_models(models_package)` in `env.py` to ensure all `BaseModel` subclasses are registered before `Base.metadata` is read. [48](#0-47)

### Citations

**File:** README.md (L1-9)
```markdown
# HexCore [![PyPI Downloads](https://static.pepy.tech/personalized-badge/hexcore?period=total&units=INTERNATIONAL_SYSTEM&left_color=BLACK&right_color=GREEN&left_text=downloads)](https://pepy.tech/projects/hexcore)
HexCore es un módulo base reutilizable para proyectos Python que implementan arquitectura hexagonal y event handling.

## ¿Qué provee HexCore?

- **Clases base y abstracciones** para entidades, repositorios, servicios y unidad de trabajo (UoW), siguiendo los principios de DDD y arquitectura hexagonal.
- **Interfaces y contratos** para caché, eventos y manejo de dependencias, desacoplando la lógica de negocio de la infraestructura.
- **Utilidades para event sourcing y event dispatching** listas para usar en cualquier proyecto.
- **Estructura flexible** para que puedas construir microservicios o aplicaciones monolíticas desacopladas y testeables.
```

**File:** pyproject.toml (L1-23)
```text
[project]
name = "hexcore"
version = "1.3.2"
authors = [
  { name="David Latosefki (Indroic)", email="indroic@outlook.com" },
]
description = "Núcleo reutilizable para proyectos Python con arquitectura hexagonal y event handling. Provee abstracciones, utilidades y contratos para DDD, eventos y desacoplamiento de infraestructura."
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "aiosqlite>=0.21.0",
    "alembic>=1.16.5",
    "async-typer>=0.1.10",
    "asyncpg>=0.30.0",
    "beanie>=2.0.0",
    "fastapi>=0.116.1",
    "pika>=1.3.2",
    "pydantic>=2.11.7",
    "redis>=6.4.0",
    "sqlalchemy>=2.0.43",
    "typer>=0.17.3",
    "ruff>=0.12.11",
]
```

**File:** DOCS.md (L1-33)
```markdown
# Documentación HexCore: Estructura, Documentos, Repositorios y Abstracciones

## 1. Estructura de Directorios y Documentos

HexCore organiza el código siguiendo los principios DDD y arquitectura hexagonal. Los principales componentes son:

- **src/domain/**  
  Módulos de dominio, cada uno con entidades, repositorios, servicios, objetos de valor, eventos, enums y excepciones.
  ```
  src/domain/{modulo}/
    ├─ __init__.py
    ├─ entities.py
    ├─ repositories.py
    ├─ services.py
    ├─ value_objects.py
    ├─ events.py
    ├─ enums.py
    └─ exceptions.py
  ```

- **src/application/**  
  Casos de uso y DTOs para orquestar la lógica de negocio.

- **src/infrastructure/**  
  Implementaciones técnicas: ORM/ODM, CLI, caché, base de datos, repositorios, unit of work.

- **src/infrastructure/database/**
  - **models/**: Modelos SQLAlchemy (relacional).
  - **documents/**: Documentos Beanie (ODM para MongoDB).

- **tests/**  
  Pruebas para cada módulo de dominio e infraestructura.

```

**File:** hexcore/config.py (L14-54)
```python
class ServerConfig(BaseModel):
    # Project Config
    base_dir: Path = Path(".")

    # SERVER CONFIG
    host: str = "localhost"
    port: int = 8000
    debug: bool = True

    # DB CONFIG
    sql_database_url: str = "sqlite:///./db.sqlite3"
    async_sql_database_url: str = "sqlite+aiosqlite:///./db.sqlite3"

    mongo_database_url: str = "mongodb://localhost:27017"
    async_mongo_database_url: str = "mongodb+async://localhost:27017"
    mongo_db_name: str = "euphoria_db"
    mongo_uri: str = "mongodb://localhost:27017/euphoria_db"

    redis_uri: str = "redis://localhost:6379/0"
    redis_host: str = "localhost"
    redis_port: int = 6379
    redis_db: int = 0
    redis_cache_duration: int = 300  # seconds

    # Security
    allow_origins: list[str] = [
        "*" if debug else "http://localhost:{port}".format(port=port)
    ]
    allow_credentials: bool = True
    allow_methods: list[str] = ["*"]
    allow_headers: list[str] = ["*"]

    # caching
    cache_backend: ICache = (
        MemoryCache()
    )  # Debe ser una instancia de ICache(o subclase)

    model_config = ConfigDict(arbitrary_types_allowed=True)

    # Event Dispatcher
    event_dispatcher: IEventDispatcher = InMemoryEventDispatcher()
```

**File:** hexcore/config.py (L57-99)
```python
class LazyConfig:
    """
    Loader de configuración flexible.
    Busca una variable 'config' (instancia de ServerConfig) o una clase 'ServerConfig' en los módulos personalizados.
    Si no la encuentra, usa la configuración base del kernel.

    IMPORTANTE: La configuración personalizada debe estar en un módulo llamado 'config' en src.domain

    """

    _imported_config: t.Optional[ServerConfig] = None

    @classmethod
    def get_config(cls) -> ServerConfig:
        if cls._imported_config is not None:
            return cls._imported_config
        # Intenta importar la config personalizada
        for modpath in ("config", "src.domain.config"):
            try:
                mod = importlib.import_module(modpath)
                config_instance = getattr(mod, "config", None)
                if config_instance is not None:
                    # Si es clase, instanciar
                    if isinstance(config_instance, type) and issubclass(
                        config_instance, ServerConfig
                    ):
                        config_instance = config_instance()
                    if isinstance(config_instance, ServerConfig):
                        cls._imported_config = config_instance
                        return cls._imported_config
                # Alternativamente, busca la clase ServerConfig
                config_class = getattr(mod, "ServerConfig", None)
                if isinstance(config_class, type) and issubclass(
                    config_class, ServerConfig
                ):
                    config_instance = config_class()
                    cls._imported_config = config_instance
                    return cls._imported_config
            except (ModuleNotFoundError, AttributeError):
                continue
        # Fallback: config base del kernel
        cls._imported_config = ServerConfig()
        return cls._imported_config
```

**File:** hexcore/domain/base.py (L12-59)
```python
class BaseEntity(BaseModel):
    """
    Clase base para todas las entidades del dominio.

    Proporciona campos comunes y configuración de Pydantic para asegurar consistencia
    y comportamiento predecible en todo el modelo.

    Atributos:
        id (UUID): Identificador único universal para la entidad.
        created_at (datetime): Marca de tiempo de la creación de la entidad (UTC).
        updated_at (datetime): Marca de tiempo de la última actualización (UTC).
        is_active (bool): Indicador para borrado lógico (soft delete).
    """

    id: UUID = Field(default_factory=uuid4)
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    is_active: t.Optional[bool] = True

    _domain_events: t.List[DomainEvent] = []

    model_config = ConfigDict(
        from_attributes=True,  # Permite crear modelos desde atributos de objetos (clave para ORMs).
        validate_assignment=True,  # Vuelve a validar la entidad cada vez que se modifica un campo.
        # `frozen=True` hace que las entidades sean inmutables, lo cual es un ideal de DDD.
        # Sin embargo, puede complicar el manejo de estado con un ORM como SQLAlchemy,
        # donde los objetos a menudo se modifican y luego se guardan.
        # Lo cambiamos a False para un enfoque más pragmático.
        frozen=False,
    )

    def register_event(self, event: DomainEvent) -> None:
        """Añade un evento a la lista de la entidad."""
        self._domain_events.append(event)

    def pull_domain_events(self) -> t.List[DomainEvent]:
        """Entrega los eventos y limpia la lista."""
        events = self._domain_events[:]
        self._domain_events.clear()
        return events

    def clear_domain_events(self) -> None:
        """Limpia la lista de eventos sin entregarlos."""
        self._domain_events.clear()

    async def deactivate(self) -> None:
        """Desactiva la Entidad(Borrado Logico)"""
        self.is_active = False
```

**File:** hexcore/domain/base.py (L62-80)
```python
class AbstractModelMeta(BaseEntity, abc.ABC):
    """
    Metaclase para resolver un conflicto entre Pydantic y las clases abstractas de Python.

    Problema:
        - Pydantic (`BaseModel`) usa su propia metaclase para la validación de datos.
        - Las clases abstractas de Python (`abc.ABC`) usan `abc.ABCMeta` para permitir `@abstractmethod`.
        - Una clase no puede tener dos metaclases diferentes.

    Solución:
        Esta metaclase combina ambas, permitiendo crear clases que son a la vez
        modelos de Pydantic y clases base abstractas.

    Uso:
        class MiClaseAbstracta(BaseEntity, abc.ABC, metaclass=AbstractModelMeta):
            ...
    """

    pass
```

**File:** hexcore/domain/events.py (L1-70)
```python
from __future__ import annotations
import typing as t
import abc
from datetime import datetime, UTC
from uuid import UUID, uuid4
from pydantic import BaseModel, Field, ConfigDict, computed_field

from .base import BaseEntity

T = t.TypeVar("T", bound=BaseEntity)


class DomainEvent(BaseModel):
    """
    Clase base abstracta para todos los eventos de dominio.
    Los eventos de dominio representan algo significativo que ha ocurrido en el dominio.
    """

    # Identificador único del evento
    event_id: UUID = Field(default_factory=uuid4)
    # Marca de tiempo de cuándo ocurrió el evento
    occurred_on: datetime = Field(default_factory=lambda: datetime.now(UTC))

    @computed_field
    @property
    def event_name(self) -> str:
        """Nombre de la clase del evento, usado para serialización/deserialización."""
        return self.__class__.__name__.replace("Event", "").upper()

    model_config = ConfigDict(
        from_attributes=True,
        frozen=True,  # Los eventos de dominio son inmutables
    )


class EntityCreatedEvent(DomainEvent, t.Generic[T]):
    """Evento base para cuando una entidad es creada."""

    entity_id: UUID
    entity_data: T


class EntityUpdatedEvent(DomainEvent, t.Generic[T]):
    """Evento base para cuando una entidad es actualizada."""

    entity_id: UUID
    entity_data: T


class EntityDeletedEvent(DomainEvent):
    """Evento base para cuando una entidad es eliminada."""

    entity_id: UUID


EventHandler = t.Callable[[DomainEvent], t.Awaitable[None]]


class IEventDispatcher(abc.ABC):
    """
    Interfaz (Puerto) para el despachador de eventos.
    """

    @abc.abstractmethod
    def register(self, event_type: type, handler: EventHandler) -> None:
        raise NotImplementedError

    @abc.abstractmethod
    async def dispatch(self, event: t.Any) -> None:
        raise NotImplementedError
```

**File:** hexcore/domain/repositories.py (L12-41)
```python
class IBaseRepository(abc.ABC, t.Generic[T]):
    """
    Interfaz genérica para un repositorio base.
    Define las operaciones CRUD comunes que todos los repositorios deben implementar.
    """

    def __init__(self, uow: IUnitOfWork):
        self.uow: IUnitOfWork = uow

    @abc.abstractmethod
    async def get_by_id(self, entity_id: UUID) -> T:
        raise NotImplementedError

    async def get_active_by_id(self, entity_id: UUID) -> T:
        entity = await self.get_by_id(entity_id)
        if not entity.is_active:
            raise InactiveEntityException
        return entity

    @abc.abstractmethod
    async def list_all(self) -> t.List[T]:
        raise NotImplementedError

    @abc.abstractmethod
    async def save(self, entity: T) -> T:
        raise NotImplementedError

    @abc.abstractmethod
    async def delete(self, entity: T) -> None:
        raise NotImplementedError
```

**File:** hexcore/domain/uow.py (L9-55)
```python
class IUnitOfWork(abc.ABC):
    """
    Interfaz para la Unidad de Trabajo. Define un contexto transaccional.
    """

    def __init__(self):
        self.repositories: t.Dict[str, t.Any] = {}
        self.events_dispatcher: t.Any = None

    async def __aenter__(self) -> IUnitOfWork:
        return self

    async def __aexit__(
        self,
        exc_type: t.Optional[type],
        exc_val: t.Optional[BaseException],
        exc_tb: t.Optional[t.Any],
    ) -> None:
        await self.rollback()

    @abc.abstractmethod
    def IUnitOfWork(self) -> t.Set[BaseEntity]:
        raise NotImplementedError

    @abc.abstractmethod
    def collect_domain_events(self) -> t.List[t.Any]:
        raise NotImplementedError

    @abc.abstractmethod
    async def dispatch_events(self) -> None:
        raise NotImplementedError

    @abc.abstractmethod
    def clear_tracked_entities(self) -> None:
        raise NotImplementedError

    @abc.abstractmethod
    async def commit(self) -> None:
        raise NotImplementedError

    @abc.abstractmethod
    async def rollback(self) -> None:
        raise NotImplementedError

    @abc.abstractmethod
    def collect_entity(self, entity: BaseEntity) -> None:
        raise NotImplementedError
```

**File:** hexcore/domain/services.py (L1-11)
```python
from hexcore.domain.events import IEventDispatcher
from hexcore.config import LazyConfig


class BaseDomainService:
    def __init__(
        self,
        event_dispatcher: IEventDispatcher = LazyConfig().get_config().event_dispatcher,
    ) -> None:
        self.config = LazyConfig.get_config()
        self.event_dispatcher = event_dispatcher
```

**File:** hexcore/domain/exceptions.py (L1-5)
```python
class InactiveEntityException(Exception):
    """Excepción lanzada cuando se intenta operar con una entidad inactiva."""

    def __init__(self) -> None:
        super().__init__("La entidad no está activa.")
```

**File:** hexcore/domain/auth/permissions.py (L28-71)
```python
class PermissionsRegistry:
    """
    Clase para registrar y consultar permisos de forma dinámica.
    Cada instancia mantiene su propio registro de permisos.
    """
    def __init__(self):
        self._permissions_registry: Dict[str, str] = {}

    def register_permission(self, name: str, value: Optional[str] = None) -> None:
        """
        Registra un nuevo permiso en el sistema.
        name: nombre identificador (ej: 'USERS_INVITE')
        value: valor string (ej: 'users.invite'). Si no se pasa, se usa name.lower().
        """
        if value is None:
            value = name.lower()
        self._permissions_registry[name] = value

    def register_permissions(self, permissions: Dict[str, Optional[str]]) -> None:
        """
        Registra múltiples permisos en el sistema.
        """
        for name, value in permissions.items():
            self.register_permission(name, value)

    def get_permissions_registry(self) -> Dict[str, str]:
        """Devuelve el registro actual de permisos (nombre: valor)."""
        return dict(self._permissions_registry)

    def get_all_permission_values(self) -> Set[str]:
        """
        Devuelve un conjunto con todos los valores de los permisos registrados.
        """
        return set(self._permissions_registry.values())

    def get_permission_by_name(self, name: str) -> Optional[str]:
        """Devuelve el valor del permiso por su nombre."""
        return self._permissions_registry.get(name)

    def build_permissions_enum(self) -> Enum:
        """
        Construye un Enum dinámico con los permisos actuales.
        """
        return Enum("PermissionsEnum", {k: v for k, v in self._permissions_registry.items()}, type=str)
```

**File:** hexcore/domain/auth/value_objects.py (L8-17)
```python
class TokenClaims(BaseModel):
    """Detalles sobre los claims de un token."""

    iss: str  # Identificador del token de acceso
    sub: str  # ID del usuario
    exp: int  # Tiempo de expiración
    iat: int  # Tiempo de emisión
    jti: str = Field(default_factory=lambda: str(uuid4()))  # ID del token
    client_id: str  # ID del cliente OAuth
    scopes: t.List[Enum] = []  # Permisos del Token
```

**File:** hexcore/application/dtos/base.py (L1-12)
```python
from abc import ABC
from pydantic import BaseModel


class DTO(BaseModel, ABC):
    """
    Clase base para todos los DTOs de la capa de aplicación.

    Estos representan los datos que se desea exponer o recibir a través de la API,
    evitando la exposición de detalles internos del dominio.
    """

```

**File:** hexcore/application/use_cases/base.py (L1-15)
```python
import typing as t
from abc import ABC, abstractmethod
from hexcore.application.dtos.base import DTO

# Tipo del Input
T = t.TypeVar("T", bound=DTO)

# Tipo del Output(o resultado)
R = t.TypeVar("R", bound=DTO)


class UseCase(ABC, t.Generic[T, R]):
    @abstractmethod
    async def execute(self, command: T) -> R:
        raise NotImplementedError("Subclasses must implement this method")
```

**File:** hexcore/infrastructure/repositories/orms/sqlalchemy/__init__.py (L15-43)
```python
class Base(DeclarativeBase):
    pass


class BaseModel(Base, t.Generic[T]):
    __abstract__ = True
    __tablename__ = "base_model"

    id: Mapped[PythonUUID] = mapped_column(UUID, primary_key=True, default=uuid4)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=lambda: datetime.now(UTC)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(UTC),
        onupdate=lambda: datetime.now(UTC),
    )

    _domain_entity: T

    def set_domain_entity(self, entity: T) -> None:
        self._domain_entity = entity

    def get_domain_entity(self) -> T:
        return self._domain_entity

    def __repr__(self):
```

**File:** hexcore/infrastructure/repositories/orms/sqlalchemy/session.py (L1-22)
```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from hexcore.config import LazyConfig


# 1. CREAR EL ENGINE ASÍNCRONO DE SQLAlchemy
# Usamos create_async_engine en lugar de create_engine.
engine = create_async_engine(
    LazyConfig.get_config().async_sql_database_url,
    # `echo=True` es útil para depuración, ya que imprime todas las sentencias SQL.
    # Desactívalo en producción.
    # echo=True,
)

# 2. CREAR UNA FACTORÍA DE SESIONES ASÍNCRONAS
# Usamos async_sessionmaker y especificamos la clase AsyncSession.
AsyncSessionLocal: async_sessionmaker[AsyncSession] = async_sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
    class_=AsyncSession,
)
```

**File:** hexcore/infrastructure/repositories/orms/sqlalchemy/session.py (L25-32)
```python
async def get_async_db_session():
    """Generador de dependencias para obtener una sesión de BD asíncrona."""
    db = AsyncSessionLocal()
    try:
        yield db
    finally:
        await db.close()

```

**File:** hexcore/infrastructure/repositories/orms/beanie/__init__.py (L8-20)
```python
class BaseDocument(Document):
    entity_id: t.Annotated[UUID, Indexed(unique=True)]
    created_at: t.Optional[datetime] = datetime.now()
    updated_at: t.Optional[datetime] = datetime.now()
    is_active: t.Optional[bool] = True

    class Settings:
        is_root = True
        use_cache = True

    @after_event([Save])
    def update_updated_at(self):
        self.updated_at = datetime.now()
```

**File:** hexcore/infrastructure/repositories/base.py (L13-23)
```python
class BaseSQLAlchemyRepository(IBaseRepository[T], abc.ABC, t.Generic[T]):
    def __init__(self, uow: IUnitOfWork):
        self._session: t.Optional[AsyncSession] = getattr(uow, "session", None)

        super().__init__(uow)
        
    @property
    def session(self) -> AsyncSession:
        if self._session is None:
            raise ValueError("El repositorio no está asociado a una sesión de base de datos.")
        return self._session
```

**File:** hexcore/infrastructure/repositories/implementations.py (L36-58)
```python
class HasBasicArgs(t.Generic[T, A]):
    @property
    def entity_cls(self) -> t.Type[T]:
        raise NotImplementedError("Debe implementar la propiedad entity_cls")

    @property
    def not_found_exception(self) -> t.Type[Exception]:
        raise NotImplementedError("Debe implementar la propiedad not_found_exception")

    @property
    def fields_serializers(self) -> FieldSerializersType[T]:
        """
        Serializadores para campos complejos en la conversión entre Entidad -> Documento/Modelo.
        """
        return {}

    @property
    def fields_resolvers(self) -> FieldResolversType[A]:
        """
        Resolvedores para campos complejos en la conversión entre Documento/Modelo -> Entidad.
        Debe ser implementado por cada repositorio específico.
        """
        return {}
```

**File:** hexcore/infrastructure/repositories/implementations.py (L61-125)
```python
class SQLAlchemyCommonImplementationsRepo(
    BaseSQLAlchemyRepository[T], HasBasicArgs[T, M], t.Generic[T, M]
):
    """
    Implementaciones comunes para repositorios SQL usando SQLAlchemy.
    Proporciona métodos CRUD genéricos que pueden ser reutilizados por repositorios específicos.
    Requiere que se especifiquen las clases de entidad y modelo, así como la excepción a lanzar cuando no se encuentra una entidad.
    - entity_cls: La clase de la entidad de dominio.
    - model_cls: La clase del modelo SQLAlchemy.
    - not_found_exception: La excepción a lanzar cuando no se encuentra una entidad.
    - uow: La unidad de trabajo para manejar transacciones y sesiones.
    Ejemplo de uso:
        class UserRepository(IUserRepo, SqlCommonImplementationsRepo[UserEntity, UserModel]):
            def __init__(self, uow: SqlAlchemyUnitOfWork):
                IUserRepo.__init__(self, uow)
                SqlCommonImplementationsRepo.__init__(self, UserEntity, UserModel, UserNotFoundException, uow)

            async def get_by_email(self, email: str) -> UserEntity:
                # Implementación específica del repositorio


    """

    @property
    def model_cls(self) -> t.Type[M]:
        raise NotImplementedError("Debe implementar la propiedad document_cls")

    async def get_by_id(self, entity_id: UUID) -> T:

        model = await sql_db_get(
            self.session,
            self.model_cls,
            entity_id,
            self.not_found_exception(entity_id),
        )
        return await to_entity_from_model_or_document(
            model, self.entity_cls, self.fields_resolvers
        )

    async def list_all(self) -> t.List[T]:

        models = await sql_db_list(self.session, self.model_cls)
        return [
            await to_entity_from_model_or_document(
                model, self.entity_cls, self.fields_resolvers
            )
            for model in models
        ]

    async def save(self, entity: T) -> T:

        saved = await sql_save_entity(
            self.session,
            entity,
            self.model_cls,
            fields_serializers=self.fields_serializers,
        )
        return await to_entity_from_model_or_document(
            saved, self.entity_cls, self.fields_resolvers
        )

    async def delete(self, entity: T) -> None:

        await sql_logical_delete(self.session, entity, self.model_cls)

```

**File:** hexcore/infrastructure/repositories/implementations.py (L127-179)
```python
class BeanieODMCommonImplementationsRepo(
    IBaseRepository[T], HasBasicArgs[T, D], t.Generic[T, D]
):
    """
    Implementaciones comunes para repositorios usando Beanie.
    Proporciona métodos CRUD genéricos que pueden ser reutilizados por repositorios específicos.
    Requiere que se especifiquen las clases de entidad y documento, así como la excepción a lanzar cuando no se encuentra una entidad.
    - entity_cls: La clase de la entidad de dominio.
    - document_cls: La clase del documento Beanie.
    - not_found_exception: La excepción a lanzar cuando no se encuentra una entidad.
    - fields_resolvers: Resolvedores para campos complejos en la conversión entre entidad y documento.
    Ejemplo de uso:
        class UserRepository(IUserRepo, NoSQLCommonImplementationsRepo[UserEntity]):
            def __init__(self, uow: NoSqlUnitOfWork):
                IUserRepo.__init__(self, uow)
                NoSQLCommonImplementationsRepo.__init__(self, UserEntity, UserDocument, UserNotFoundException, uow)

            async def get_by_email(self, email: str) -> UserEntity:
                # Implementación específica del repositorio
    """

    @property
    def document_cls(self) -> t.Type[D]:
        raise NotImplementedError("Debe implementar la propiedad document_cls")

    async def get_by_id(self, entity_id: UUID) -> T:
        document = await nosql_db_get(self.document_cls, entity_id)
        if not document:
            raise self.not_found_exception(entity_id)
        return await to_entity_from_model_or_document(
            document, self.entity_cls, self.fields_resolvers, is_nosql=True
        )

    async def list_all(self) -> t.List[T]:
        documents = await nosql_db_list(self.document_cls)
        return [
            await to_entity_from_model_or_document(
                doc, self.entity_cls, self.fields_resolvers, is_nosql=True
            )
            for doc in documents
        ]

    @register_entity_on_uow
    async def save(self, entity: T) -> T:  # type: ignore
        saved = await nosql_save_entity(
            entity, self.document_cls, self.fields_serializers
        )
        return await to_entity_from_model_or_document(
            saved, self.entity_cls, self.fields_resolvers, is_nosql=True
        )

    async def delete(self, entity: T) -> None:
        return await nosql_logical_delete(entity.id, self.document_cls)
```

**File:** hexcore/infrastructure/uow/__init__.py (L13-77)
```python
class SqlAlchemyUnitOfWork(IUnitOfWork):
    """
    Implementación concreta (Adaptador) de la Unidad de Trabajo para SQLAlchemy.
    """

    def __init__(self, session: AsyncSession):
        self.session = session
        super().__init__()

    async def __aexit__(
        self,
        exc_type: t.Optional[type],
        exc_val: t.Optional[BaseException],
        exc_tb: t.Optional[t.Any],
    ) -> None:
        # Llama al rollback de la interfaz (que a su vez llama al nuestro)
        # y cierra la sesión para liberar la conexión.
        await super().__aexit__(exc_type, exc_val, exc_tb)
        await self.session.close()

    async def commit(self):
        """
        Confirma la transacción y despacha los eventos.
        """
        await self.session.commit()
        await self.dispatch_events()
        self.clear_tracked_entities()

    async def rollback(self):
        await self.session.rollback()
        self.clear_tracked_entities()

    def collect_domain_entities(self) -> t.Set[BaseEntity]:
        """
        Recolecta todas las entidades de dominio rastreadas por la sesión de SQLAlchemy.
        """
        domain_entities: t.Set[BaseEntity] = set()
        all_tracked_models = chain(
            self.session.new, self.session.dirty, self.session.deleted
        )
        for model in all_tracked_models:
            if isinstance(model, BaseModel):
                entity: BaseEntity = model.get_domain_entity()  # type: ignore
                assert isinstance(entity, BaseEntity)
                domain_entities.add(entity)
        return domain_entities

    def collect_domain_events(self) -> t.List[DomainEvent]:
        events: t.List[DomainEvent] = []
        for entity in self.collect_domain_entities():
            events.extend(entity.pull_domain_events())
        return events

    async def dispatch_events(self):
        for event in self.collect_domain_events():
            await self.events_dispatcher.dispatch(event)

    def clear_tracked_entities(self):
        # No es necesario limpiar entidades en SQL, pero se mantiene para simetría
        pass

    def collect_entity(self, entity: BaseEntity) -> None:
        # No es necesario en SQLAlchemy, pero se define para compatibilidad
        pass

```

**File:** hexcore/infrastructure/uow/__init__.py (L79-121)
```python
class NoSqlUnitOfWork(IUnitOfWork):
    def __init__(self):
        self._entities: set[BaseEntity] = set()

    async def __aenter__(self):
        return self

    async def __aexit__(
        self,
        exc_type: t.Optional[type],
        exc_val: t.Optional[BaseException],
        exc_tb: t.Optional[t.Any],
    ) -> None:
        if exc_type:
            await self.rollback()

    async def commit(self):
        await self.dispatch_events()
        self.clear_tracked_entities()

    async def rollback(self):
        for entity in self._entities:
            entity.clear_domain_events()
        self.clear_tracked_entities()

    def collect_entity(self, entity: BaseEntity):
        self._entities.add(entity)

    def collect_domain_entities(self) -> t.Set[BaseEntity]:
        return set(self._entities)

    def collect_domain_events(self) -> t.List[DomainEvent]:
        events: t.List[DomainEvent] = []
        for entity in self.collect_domain_entities():
            events.extend(entity.pull_domain_events())
        return events

    async def dispatch_events(self):
        for event in self.collect_domain_events():
            await self.events_dispatcher.dispatch(event)

    def clear_tracked_entities(self):
        self._entities.clear()
```

**File:** hexcore/infrastructure/api/utils.py (L1-20)
```python
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends

from hexcore.infrastructure.repositories.orms.sqlalchemy.session import AsyncSessionLocal
from hexcore.infrastructure.uow import SqlAlchemyUnitOfWork, NoSqlUnitOfWork


async def get_session():
    async with AsyncSessionLocal() as session:
        yield session


async def get_sql_uow(session: AsyncSession = Depends(get_session)):
    async with SqlAlchemyUnitOfWork(session=session) as uow:
        yield uow


async def get_nosql_uow():
    async with NoSqlUnitOfWork() as uow:
        yield uow
```

**File:** hexcore/infrastructure/cache/__init__.py (L1-17)
```python
import typing as t
import abc


class ICache(abc.ABC):
    @abc.abstractmethod
    async def get(self, key: str) -> t.Optional[t.Any]:
        pass

    @abc.abstractmethod
    def set(self, key: str, value: t.Any, expire: int = 3600) -> t.Any:
        pass

    @abc.abstractmethod
    def delete(self, key: str) -> t.Union[t.Any, None]:
        pass

```

**File:** hexcore/infrastructure/cache/cache_backends/memory.py (L5-24)
```python
class MemoryCache(ICache):
    def __init__(self):
        self.cache: t.Dict[str, t.Dict[str, t.Any]] = {}

    async def get(self, key: str) -> t.Optional[t.Dict[str, t.Any]]:
        return self.cache.get(key)

    async def set(
        self,
        key: str,
        value: t.Dict[str, t.Any],
        expire: int = 0,
    ) -> None:
        self.cache[key] = value

    async def delete(self, key: str) -> None:
        self.cache.pop(key, None)

    async def clear(self) -> None:
        self.cache.clear()
```

**File:** hexcore/infrastructure/cache/cache_backends/redis.py (L8-30)
```python
class RedisCache(ICache):
    def __init__(self):
        config = LazyConfig.get_config()
        self.redis: redis.Redis = redis.Redis.from_url(  # type: ignore
            config.redis_uri, decode_responses=True
        )

    async def get(self, key: str) -> t.Optional[t.Dict[str, t.Any]]:
        value = await self.redis.get(key)
        return json.loads(value) if value else None

    async def set(
        self,
        key: str,
        value: t.Dict[str, t.Any],
        expire: int = LazyConfig().get_config().redis_cache_duration,
    ) -> None:
        await self.redis.set(key, json.dumps(value), ex=expire)

    async def delete(self, key: str) -> None:
        await self.redis.delete(key)

    async def clear(self):
```

**File:** hexcore/infrastructure/events/events_backends/memory.py (L5-20)
```python
class InMemoryEventDispatcher(IEventDispatcher):
    def __init__(self) -> None:
        self._events: list[tuple[str, dict[str, t.Any]]] = []

    async def dispatch(self, event: DomainEvent) -> None:
        self._events.append((event.__class__.__name__, event.model_dump()))

    def register(
        self,
        event_type: t.Type[DomainEvent],
        handler: t.Callable[[DomainEvent], t.Awaitable[None]],
    ) -> None:
        pass

    def clear_events(self) -> None:
        self._events.clear()
```

**File:** hexcore/types.py (L1-36)
```python
import typing as t
from typing import Protocol
from .domain.base import BaseEntity
from .infrastructure.repositories.orms.sqlalchemy import BaseModel
from .infrastructure.repositories.base import IBaseRepository
from uuid import UUID

# Tipos genéricos reutilizables para repositorios y decoradores
A = t.TypeVar("A", contravariant=True)
T = t.TypeVar("T", bound=BaseEntity)
SelfRepoT = t.TypeVar("SelfRepoT", bound=IBaseRepository[t.Any])
EntityT = t.TypeVar("EntityT", bound=BaseEntity)
P = t.ParamSpec("P")
R = t.TypeVar("R")


# Protocolo para resolvers asíncronos que aceptan visited como kwarg
class AsyncResolver(Protocol[A]):
    async def __call__(
        self, model: A, *, visited: t.Optional[set[str]] = None, **kwargs: t.Any
    ) -> t.Any: ...


# Tipos para los parámetros de contexto de protección de ciclo
VisitedType: t.TypeAlias = set[int]
VisitedResultsType: t.TypeAlias = dict[int, t.Any]
# Tipo para la firma de los resolvedores asíncronos con protección de ciclo (solo model)
AsyncCycleResolver: t.TypeAlias = t.Callable[[A], t.Awaitable[t.Any]]

FieldResolversType: t.TypeAlias = t.Dict[str, t.Tuple[str, AsyncCycleResolver[A]]]


FieldSerializersType: t.TypeAlias = t.Dict[str, t.Tuple[str, t.Callable[[A], t.Any]]]

ExcludeType: t.TypeAlias = t.Optional[set[str]]

```

**File:** hexcore/infrastructure/repositories/decorators.py (L14-48)
```python
def cycle_protection_resolver(
    func: Callable[[A], Awaitable[Any]],
) -> Callable[[A], Awaitable[Any]]:
    """
    Decorador para resolvedores asíncronos que protege contra recursividad infinita en relaciones cíclicas.

    Mecanismo:
        1. Cada vez que se llama al resolvedor decorado, se crean dos estructuras internas:
            - visited: un set que almacena los ids de las entidades ya visitadas en la cadena de resolución actual.
            - visited_results: un diccionario que almacena el resultado ya calculado para cada id de entidad.
        2. Cuando el resolvedor es llamado con una entidad:
            - Si el id de la entidad ya está en visited, significa que se está entrando en un ciclo. En ese caso, el decorador retorna el resultado previamente calculado para ese id (o None si no existe), evitando así la recursión infinita.
            - Si no está en visited, agrega el id y ejecuta el resolvedor normalmente.
            - Al terminar, almacena el resultado en visited_results para ese id.
        3. Así, aunque los resolvedores se llamen recursivamente (por ejemplo, en relaciones A → B → A), nunca se entra en un bucle infinito porque el decorador corta la recursión y retorna el valor ya calculado.
    """

    async def wrapper(model: A) -> Any:
        entity_id = getattr(model, "id", None)
        visited = _visited_ctx.get().copy()
        results = _results_ctx.get().copy()
        token_visited = _visited_ctx.set(visited)
        token_results = _results_ctx.set(results)
        try:
            if entity_id is not None:
                if entity_id in visited:
                    return results.get(entity_id)
                visited.add(entity_id)
            result = await func(model)
            if entity_id is not None:
                results[entity_id] = result
            return result
        finally:
            _visited_ctx.reset(token_visited)
            _results_ctx.reset(token_results)
```

**File:** hexcore/infrastructure/uow/decorators.py (L6-14)
```python
def register_entity_on_uow(
    method: t.Callable[[SelfRepoT, EntityT], t.Awaitable[EntityT]],
) -> t.Callable[[SelfRepoT, EntityT], t.Awaitable[EntityT]]:
    @wraps(method)
    async def wrapper(self: SelfRepoT, entity: EntityT) -> EntityT:
        result = await method(self, entity)
        if getattr(entity, "_domain_events", None):
            self.uow.collect_entity(entity)
        return result
```

**File:** hexcore/infrastructure/repositories/utils.py (L57-62)
```python
    for field, (data_field, resolver) in field_resolvers.items():
        # Si el campo existe en el dict
        if data_field in model_dict:
            # Llama al resolvedor pasando solo el modelo
            model_dict[field] = await resolver(model_or_doc)
    # Devuelve el dict con los campos resueltos
```

**File:** hexcore/infrastructure/repositories/utils.py (L66-91)
```python
async def to_entity_from_model_or_document(
    model_instance: T,
    entity_class: t.Type[E],
    field_resolvers: t.Optional[FieldResolversType[T]] = None,
    is_nosql: bool = False,
) -> E:
    """
    Convierte un modelo SQLAlchemy o un documento Beanie a una entidad de dominio,
    permitiendo reconstruir campos complejos con resolvers asíncronos.
    Si is_nosql=True, renombra 'entity_id' a 'id'.
    
    SOLO FUNCIONA CON LOS ORMS/ODMS SOPORTADOS (SQLAlchemy Y BEANIE).
    """
    model_dict = (
        model_instance.model_dump()
        if is_nosql and isinstance(model_instance, BaseDocument)
        else model_instance.__dict__.copy()
    )
    if is_nosql and "entity_id" in model_dict:
        model_dict["id"] = model_dict.pop("entity_id")
    model_dict = await _apply_async_field_resolvers(
        model_instance, model_dict, field_resolvers
    )
    if is_nosql:
        return entity_class.model_validate(model_dict)
    return entity_class.model_validate(model_dict, from_attributes=True)
```

**File:** hexcore/infrastructure/repositories/utils.py (L103-118)
```python
def discover_sql_repositories() -> t.Dict[
    str,
    t.Type[BaseSQLAlchemyRepository[t.Any]],
]:
    """
    Descubre todos los repositorios SQL disponibles.

    Retorna un diccionario que mapea nombres de repositorios a sus clases.
    El nombre del repositorio se deriva del nombre de la clase, convirtiéndolo a minúsculas.
    Ejemplo:
        Si existe una clase UserRepository, se mapeará como 'user': UserRepository
    """

    return {
        repo_cls.__name__.lower().replace("repository", ""): repo_cls
        for repo_cls in get_all_concrete_subclasses(BaseSQLAlchemyRepository)
```

**File:** hexcore/infrastructure/repositories/orms/sqlalchemy/utils.py (L21-163)
```python
def to_model(
    entity: E,
    model_cls: type[T],
    exclude: t.Optional[set[str]] = None,
    field_serializers: t.Optional[FieldResolversType[E]] = None,
    set_domain: bool = False,
) -> T:
    """
    Convierte una entidad de dominio a un modelo SQLAlchemy, permitiendo serializar campos complejos.
    - Si se especifican field_serializers, serializa campos complejos.
    - Si set_domain es True, llama a set_domain_entity en el modelo.
    """
    entity_data = entity.model_dump(exclude=exclude or set())
    if field_serializers:
        for field, (dest_field, serializer) in field_serializers.items():
            if hasattr(entity, field):
                entity_data[dest_field] = serializer(entity)
                if field in entity_data:
                    del entity_data[field]
    model = model_cls(**entity_data)
    if set_domain and hasattr(model, "set_domain_entity"):
        model.set_domain_entity(entity)
    return model


def _get_relationship_names(model: t.Type[BaseModel[t.Any]]) -> list[str]:
    return [
        key
        for key, attr in model.__mapper__.all_orm_descriptors.items()  # type: ignore
        if isinstance(getattr(attr, "property", None), RelationshipProperty)  # type: ignore
    ]


def load_relations(model: t.Type[T]) -> t.Any:
    """
    Crea una lista de opciones selectinload para las relaciones del modelo especificado.

    Args:
        model: Clase del modelo a cargar.

    Returns:
        Lista de opciones selectinload.
    """
    return [selectinload(getattr(model, rel)) for rel in _get_relationship_names(model)]


async def db_get(
    session: AsyncSession, model: t.Type[T], id: UUID, exc_none: Exception
) -> T:
    stmt = select(model).where(model.id == id)
    result = await session.execute(stmt)
    get_entity = result.scalar_one_or_none()
    if not get_entity:
        raise exc_none

    return get_entity


async def db_list(session: AsyncSession, model: t.Type[T]) -> t.List[T]:
    stmt = select(model).options(*load_relations(model))
    result = await session.execute(stmt)
    entities = list(result.scalars().all())
    if not entities:
        return []
    return entities


async def db_save(session: AsyncSession, entity: T) -> T:
    """
    Guarda una entidad en la base de datos usando merge (actualiza o inserta),
    realiza commit y refresh, y retorna la instancia gestionada.
    """
    merged = await session.merge(entity)
    await session.flush()
    await session.refresh(merged)
    return merged


def select_in_load_options(*relationships: str, model: t.Type[T]) -> t.Any:
    """
    Crea una lista de opciones selectinload para las relaciones especificadas.

    Args:
        *relationships: Nombres de las relaciones a cargar.

    Returns:
        Lista de opciones selectinload.
    """
    return [selectinload(getattr(model, rel)) for rel in relationships]


async def assign_relations(
    session: AsyncSession, model_instance: BaseModel[t.Any], relations: RelationsType
) -> None:
    for attr, (Model, ids) in relations.items():
        if ids:
            stmt = select(Model).where(Model.id.in_(ids))
            result = await session.execute(stmt)
            models = [m for m in result.scalars().all()]
            if len(models) != len(ids):
                raise ValueError(f"Uno o más {attr} especificados no existen.")
            # Asegura que todos los modelos estén en la sesión
            for m in models:
                # Verifica si el objeto está en la sesión usando get (async)
                obj_in_session = await session.get(Model, m.id)
                if not session.is_modified(m) and obj_in_session is None:
                    session.add(m)
            # Detecta si la relación es lista o única
            rel_prop = getattr(type(model_instance), attr)
            if hasattr(rel_prop, "property") and hasattr(rel_prop.property, "uselist"):
                if rel_prop.property.uselist:
                    setattr(model_instance, attr, models)
                else:
                    setattr(model_instance, attr, models[0] if models else None)
            else:
                setattr(model_instance, attr, models)


async def save_entity(
    session: AsyncSession,
    entity: E,
    model_cls: type[T],
    relations: t.Optional[RelationsType] = None,
    exclude: t.Optional[set[str]] = None,
    fields_serializers: t.Optional[FieldResolversType[E]] = None,
) -> T:
    model_instance = to_model(
        entity, model_cls, exclude, fields_serializers, set_domain=True
    )
    if relations:
        await assign_relations(session, model_instance, relations)
    saved = await db_save(session, model_instance)
    return saved


async def logical_delete(
    session: AsyncSession, entity: BaseEntity, model_cls: type[T]
) -> None:
    model = await session.get(model_cls, entity.id)
    if model:
        await entity.deactivate()
        await save_entity(session, entity, model_cls)

```

**File:** hexcore/infrastructure/repositories/orms/sqlalchemy/utils.py (L165-167)
```python
def import_all_models(package: types.ModuleType)  -> t.Any:
    for _, module_name, _ in pkgutil.iter_modules(package.__path__):
        importlib.import_module(f"{package.__name__}.{module_name}")
```

**File:** hexcore/infrastructure/repositories/orms/beanie/utils.py (L17-134)
```python
def to_document(
    entity_data: E,
    document_class: t.Type[D],
    field_serializers: t.Optional[FieldSerializersType[E]] = None,
    update: bool = False,  # Indica si se desea realizar una actualización, en ese caso No se renombrará el 'id', solo se excluirá
) -> D:
    """
    Función de ayuda para convertir una entidad de dominio a un modelo NoSQL, permitiendo serializar campos complejos.
    Args:
        entity_data: Entidad de dominio.
        document_class: Clase del documento NoSQL.
        field_serializers: Diccionario opcional {campo: (destino, serializer(entidad de dominio original))} para transformar campos complejos.
        update: Si es True, no renombra el id, solo lo excluye.
    """
    entity_data_dict = entity_data.model_dump()

    # Renombramos el 'id' de la entidad a 'entity_id' para el documento.
    if "id" in entity_data_dict and not update:
        entity_data_dict["entity_id"] = entity_data_dict.pop("id")

    # Excluimos el 'id' de la entidad en caso de actualización
    if "id" in entity_data_dict and update:
        entity_data_dict.pop("id")

    # Serializamos campos complejos y eliminamos el campo original si se especifica
    if field_serializers:
        for field, (dest_field, serializer) in field_serializers.items():
            if hasattr(entity_data, field):
                entity_data_dict[dest_field] = serializer(entity_data)
                if field in entity_data_dict:
                    del entity_data_dict[field]

    return document_class(**entity_data_dict)


def discover_beanie_documents() -> t.List[t.Type[BaseDocument]]:
    """
    Descubre todos los documentos Beanie disponibles.

    Retorna una lista de clases de documentos Beanie.
    """
    return [doc_cls for doc_cls in get_all_concrete_subclasses(BaseDocument)]


async def init_beanie_documents():
    """
    Inicializa los documentos Beanie descubiertos.
    """
    client = AsyncMongoClient(LazyConfig().get_config().mongo_uri)  # type: ignore

    documents = discover_beanie_documents()

    await init_beanie(database=client.get_default_database(), document_models=documents)


async def db_get(document_class: t.Type[D], entity_id: UUID) -> t.Optional[D]:
    """
    Obtiene un documento por su ID.
    Args:
        document_class: Clase del documento a buscar.
        id: ID del documento.
    Returns:
        El documento encontrado o None si no existe.
        :param entity_id: id de la entidad
    """
    return await document_class.find_one({"entity_id": entity_id})


async def db_list(document_class: t.Type[D]) -> t.List[D]:
    """
    Lista todos los documentos de una clase específica.
    Args:
        document_class: Clase del documento a listar.
    Returns:
        Lista de documentos encontrados.
    """
    return await document_class.find_all().to_list()


async def save_entity(
    entity: E, document_cls: t.Type[D], fields_serializers: FieldSerializersType[E]
) -> D:
    """
    Guarda o actualiza un documento en la base de datos.
    Args:
        document: Documento a guardar o actualizar.
    Returns:
        El documento guardado o actualizado.
        :param fields_resolvers: resolvers para campos complejos
        :param document_cls: Clase del Documento
        :param entity: Entidad a guardar o actualizar
    """
    document = await db_get(document_cls, entity.id)

    if document:
        # Actualización
        document = to_document(entity, document_cls, fields_serializers, update=True)
        await document.save()

        return document

    # Creación
    document = to_document(entity, document_cls, fields_serializers, update=False)
    await document.save()

    return document


async def logical_delete(entity_id: UUID, document_cls: t.Type[D]) -> None:
    """
    Realiza una eliminación lógica de un documento estableciendo is_active a False.
    Args:
        document: Documento a eliminar lógicamente.
    """
    document = await db_get(document_cls, entity_id)
    if document:
        document.is_active = False
        await document.save()
```

**File:** hexcore/infrastructure/uow/helpers.py (L1-9)
```python
import typing as t
from . import IUnitOfWork
from hexcore.infrastructure.repositories.base import IBaseRepository

T = t.TypeVar("T", bound=IBaseRepository[t.Any])


def get_repository(uow: IUnitOfWork, repo_name: str, repo_type: t.Type[T]) -> T:
    return t.cast(T, getattr(uow, repo_name))
```

**File:** hexcore/infrastructure/cli.py (L7-27)
```python
app = AsyncTyper(
    help="CLI para ayudar con tareas de desarrollo en el proyecto Euphoria."
)

# Asumiendo que el script se ejecuta desde la raíz del proyecto.
PROJECT_ROOT = LazyConfig.get_config().base_dir
DOMAIN_PATH = PROJECT_ROOT / "src" / "domain"
APPLICATION_PATH = PROJECT_ROOT / "src" / "application"
INFRAESTRUCTURE_PATH = PROJECT_ROOT / "src" / "infrastructure"
DB_PATH = INFRAESTRUCTURE_PATH / "database"

MODELS_PATH = DB_PATH / "models"
DOCUMENTS_PATH = DB_PATH / "documents"

TESTS_DOMAIN_PATH = PROJECT_ROOT / "tests" / "domain"

README = PROJECT_ROOT / "README.md"
GITIGNORE = PROJECT_ROOT / ".gitignore"
MANAGE = PROJECT_ROOT / "manage.py"


```

**File:** hexcore/infrastructure/cli.py (L306-374)
```python
def _get_repositories_template(class_name: str) -> str:
    return f"""
from __future__ import annotations
import abc
from uuid import UUID

from hexcore.domain.repositories import IBaseRepository
from .entities import {class_name}


class I{class_name}Repository(IBaseRepository[{class_name}]):
    \"\"\"
    Interfaz del repositorio para la entidad {class_name}.
    \"\"\"
    pass
"""


def _get_services_template(module_name: str, class_name: str) -> str:
    return f"""
from __future__ import annotations
import typing as t
from hexcore.domain.services import BaseDomainService

class {class_name}Service(BaseDomainService):
    \"\"\"
    Servicio de dominio para el módulo {module_name}.
    Orquesta operaciones que no encajan de forma natural en una única entidad.
    \"\"\"
    def __init__(self):
        # Los repositorios y otros servicios se inyectan aquí.
        pass

    # Define aquí los métodos del servicio
"""


def _get_value_objects_template() -> str:
    return """
from __future__ import annotations
from pydantic import BaseModel, ConfigDict
from decimal import Decimal


# Ejemplo de Objeto de Valor
# class MiObjetoDeValor(BaseModel):
#     \"\"\"
#     Un Objeto de Valor (Value Object) de ejemplo.
#     Son inmutables y se definen por sus atributos.
#     \"\"\"
#     valor: str
#
#     model_config = ConfigDict(frozen=True)
"""


def _get_events_template() -> str:
    return """
from __future__ import annotations
from hexcore.domain.events import EntityCreatedEvent


# Ejemplo de Evento de Creacion
# class MiEventoDeCreacion(EntityCreatedEvent[Entidad]):
#     \"\"\"
#     Un Evento de Creación de ejemplo.
#     \"\"\"
#   pass
"""
```

**File:** hexcore/__init__.py (L1-38)
```python
"""
Euphoria Kernel Core
Submódulo principal con entidades, eventos y repositorios.
"""

from .domain.base import BaseEntity
from .domain.auth.permissions import PermissionsRegistry
from .domain.auth.value_objects import TokenClaims
from .domain.events import (
    DomainEvent,
    EntityCreatedEvent,
    EntityDeletedEvent,
    EntityUpdatedEvent,
)
from .domain.repositories import IBaseRepository
from .infrastructure.repositories.base import (
    BaseSQLAlchemyRepository,
)
from .infrastructure import cli
from .infrastructure import cache
from .application.dtos.base import DTO
from . import config

__all__ = [
    "BaseEntity",
    "PermissionsRegistry",
    "TokenClaims",
    "DTO",
    "DomainEvent",
    "EntityCreatedEvent",
    "EntityDeletedEvent",
    "EntityUpdatedEvent",
    "IBaseRepository",
    "BaseSQLAlchemyRepository",
    "cli",
    "cache",
    "config",
]
```
