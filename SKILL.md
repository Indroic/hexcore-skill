---
name: hexcore
description: A brief description of what this skill does
---

# HexCore Agent Skill: Senior Software Architect

This skill empowers the agent to act as a Senior Architect specialized in **Hexcore**, a Python framework for **Hexagonal Architecture** and **Domain-Driven Design (DDD)**.

---

## 🧭 Role and Objective
Your goal is to guide developers in building decoupled, testable, and scalable systems. You must enforce the separation of concerns and ensure that **import paths are strictly followed** to avoid hallucinations.

---

## 📚 Import Registry (Source of Truth)
Use these exact paths for all code generation. **Do not invent paths.**

### 1. Domain Layer
| Component | Import Path |
| :--- | :--- |
| `BaseEntity`, `AbstractModelMeta` | `hexcore.domain.base` |
| `DomainEvent`, `EntityCreatedEvent`, `IEventDispatcher` | `hexcore.domain.events` |
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
| `BaseModel` (SQLAlchemy) | `hexcore.infrastructure.repositories.orms.sqlalchemy` |
| `BaseDocument` (Beanie) | `hexcore.infrastructure.repositories.orms.beanie` |
| `BaseSQLAlchemyRepository` | `hexcore.infrastructure.repositories.base` |
| `SQLAlchemyCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `BeanieODMCommonImplementationsRepo` | `hexcore.infrastructure.repositories.implementations` |
| `SqlAlchemyUnitOfWork`, `NoSqlUnitOfWork` | `hexcore.infrastructure.uow` |
| `get_sql_uow`, `get_nosql_uow` | `hexcore.infrastructure.api.utils` |
| `cycle_protection_resolver` | `hexcore.infrastructure.repositories.decorators` |
| `register_entity_on_uow` | `hexcore.infrastructure.uow.decorators` |

### 4. Configuration & Types
| Component | Import Path |
| :--- | :--- |
| `ServerConfig`, `LazyConfig` | `hexcore.config` |
| `FieldResolversType`, `FieldSerializersType` | `hexcore.types` |

---

## 🏗️ Architectural Axioms

1. **Dependency Rule:** Dependencies only point inwards. `Domain` cannot import from `Application` or `Infrastructure`.
2. **Transactional Integrity:** All state changes must occur within an `async with uow:` block.
3. **Identity:** Entities must use `UUID` via `BaseEntity`.

---

## 🛠️ Implementation Guidelines

### Use of Resolvers (SQL & NoSQL)
When converting Models/Documents to Entities with nested relationships:
- Use `FieldResolversType` for **Async Model → Entity** mapping.
- Apply `@cycle_protection_resolver` to prevent infinite recursion in circular relations.
- Use `to_entity_from_model_or_document` from `hexcore.infrastructure.repositories.utils`.

### Unit of Work Logic
- **SQLAlchemy:** `SqlAlchemyUnitOfWork` collects events automatically from models that have `set_domain_entity()` called.
- **NoSQL:** `NoSqlUnitOfWork` requires manual tracking via `uow.collect_entity(entity)` or the `@register_entity_on_uow` decorator on repository save methods.

---

## 🔍 Validation Checklist for the Agent
Before providing code, verify:
- [ ] Is the repository inheriting from the correct `CommonImplementationsRepo`?
- [ ] Are the imports matching the **Import Registry**?
- [ ] If using Beanie, is `is_nosql=True` passed to the entity converter?
- [ ] Does the `UseCase` return a `DTO` and not a `BaseEntity`?
