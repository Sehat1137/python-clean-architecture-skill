# Clean Architecture Reference

## Layer Rules

Hard rules:

- Source code dependencies point inward.
- Inner business modules must not depend on framework, transport, storage,
  dependency injection, or integration details from outer layers.
- Pydantic request/response/event models stay in presentation and are not passed
  into application or domain code.
- Application code uses infrastructure through application-owned interfaces.
- Infrastructure must not import from presentation.

Preferred conventions, unless the repository already does otherwise:

- Domain models are dataclasses, value objects, enums, or similar plain Python
  structures.
- Interactors are named after one system operation, usually
  `<Action><Entity>Interactor`.
- Place application contracts according to ownership and the repository's local
  structure. Prefer the owning use case, service, or interface over a generic
  contract module when the contract is not genuinely shared.
- Composition root owns framework setup, router registration, provider wiring,
  and runtime lifecycle.

- Domain code owns business rules, entities, value objects, enums, and domain
  errors. Do not import HTTP/MQ frameworks, database clients,
  dependency-injection markers, infrastructure code, external clients, or
  transport schemas there.
- Application code owns use cases, DTOs, interfaces, application services, and
  application errors. It may depend on domain and other application modules.
- Infrastructure code owns database tables, repository implementations,
  external adapters, and framework-specific integration.
- Presentation code owns HTTP routes, MQ/event handlers, request/response/event
  models, HTTP status mapping, broker acknowledgements, authentication headers,
  OpenAPI/AsyncAPI-facing contracts, and exception mapping.

Dependencies point inward. Interfaces belong in the inner layer where the
contract is consumed; implementations belong in outer layers.

## Target Structure For New Services

For new backend services, prefer this Clean Architecture layout:

```text
.
├── application
│   ├── dtos          # optional genuinely shared application contracts
│   ├── errors        # optional application/business workflow errors
│   ├── interactors   # application use cases
│   ├── interfaces    # ports for infrastructure and external dependencies
│   └── services      # optional shared business logic for interactors
├── domain
│   ├── entities      # application entities
│   ├── errors        # optional domain errors
│   └── services      # optional critical domain logic
├── infrastructure
│   ├── adapters      # optional adapters for external APIs/systems
│   ├── errors        # optional infrastructure errors
│   ├── repositories  # data mappers for domain entities
│   └── schema        # optional database schema definitions
└── presentation
    ├── endpoints     # optional HTTP endpoints and related models
    ├── errors        # optional presentation-layer errors
    └── handlers      # optional consumers and other transport handlers
```

Existing services may have mixed or older architecture. Add new functionality in
the style of the nearest code and do not start a broad architecture refactor
without a separate task.

## Dependency Inversion Scope

- Apply dependency inversion for outward dependencies from application logic to
  infrastructure: repositories, gateways, external clients, event publishers,
  clocks, ID generators, and similar runtime collaborators.
- Shape repository and gateway interfaces around application use cases or
  aggregates, not generic table CRUD. Concrete SQL, table definitions,
  migrations, indexes, and database compatibility belong to infrastructure.
- Presentation may import application interactors, DTOs, and errors directly.
  Adding interfaces between presentation and application usually adds indirection
  without improving the dependency direction.
- Composition root may import all layers for wiring, but business behavior should
  not live there.
- Provider modules such as `application/ioc.py` or `infrastructure/ioc.py` are
  wiring modules, not business modules. They may import the local DI framework
  when that is the repository convention.

## Interface Implementation Pattern

Define ports in the layer that consumes them, usually application. Implement the
port in the outer layer and wire the concrete class through the repository's DI
container or app factory.

```python
# application/interfaces/tenant_repository.py
from typing import Protocol
from uuid import UUID

from app.domain.tenant import Tenant


class TenantRepository(Protocol):
    async def get_by_id(self, tenant_id: UUID) -> Tenant | None: ...
    async def save(self, tenant: Tenant) -> None: ...
```

```python
# infrastructure/repository/tenant.py
from app.application import interfaces
from app.domain.tenant import Tenant


class TenantRepository(interfaces.TenantRepository):
    async def get_by_id(self, tenant_id: UUID) -> Tenant | None:
        ...

    async def save(self, tenant: Tenant) -> None:
        ...
```

Explicit inheritance from a `Protocol` is useful when the repository already
uses it for readability or static checking. Structural conformance is also valid
Python; follow the nearest local convention. The important boundary is that
application owns the interface, infrastructure owns the implementation, and
business code receives the interface rather than constructing infrastructure.

## Contract Ownership And Cohesion

Use the Common Closure Principle, the component-level form of the Single
Responsibility Principle, as the placement heuristic: keep contracts that
change for the same reason and at the same time together. Organize DTOs, result
objects, and errors by owner, not merely by technical form such as `dataclass`
or `Exception`.

### DTO Placement

- Put a use-case-specific input, result, or helper type next to its interactor.
- Put a port-specific DTO next to the application interface that consumes or
  returns it.
- Use `application/dto/` or `application/dtos/` only when a contract is
  genuinely shared by multiple use cases and has a shared reason to change. Do
  not use a generic DTO module as a collection point for unrelated contracts.
- If an application interface returns an application DTO, define that DTO in
  application, not infrastructure.
- If a repository or gateway returns a domain entity directly, do not add a DTO
  just for ceremony.
- Keep application DTOs representation-agnostic. Map `asyncpg.Record`,
  SQLAlchemy rows, external-client payloads, and similar outer representations
  inside infrastructure, then instantiate the application-owned DTO there.
  Do not put `from_record`, column-name lookup, or client-payload parsing on the
  application DTO.
- Request/response/event Pydantic models are transport schemas, not application
  DTOs. Map them to application DTOs or primitives at the presentation boundary.

### Error Placement

- Assign an error to the workflow or layer that gives the failure semantic
  meaning, not necessarily to the component that detects or raises it. The
  detector reports a condition; the semantic owner defines what that condition
  means to the system.
- Keep errors in a dedicated module within the layer or feature that gives them
  semantic meaning.
- Do not put exception classes in an interface module merely because an adapter
  can raise them. Python port signatures do not declare raised exceptions.
- When an application workflow assigns semantic meaning to an outer failure,
  define the error in the application's dedicated error module. Let
  infrastructure translate client or database failures into that error.
- When a failure is only an adapter implementation detail and application code
  does not interpret it, keep the error local to infrastructure.
- A concrete infrastructure idempotency service may raise an
  application-owned `IdempotencyKeyConflictError` when conflict is a typed
  application outcome. Infrastructure importing and raising the inner error
  follows the dependency rule; the detection site does not become the owner.
- A repository returning no record only detects absence. When a use case
  requires that resource, `ResourceNotFoundError` is normally an application
  outcome: domain cannot reject the absence of an entity that was never
  instantiated. By contrast, an invalid resource identifier or another
  intrinsic entity/value-object invariant belongs in domain.
- Put an error shared by several related use cases in a feature-specific module
  only when those workflows genuinely share its meaning and reason to change.
- Keep domain-invariant violations in domain. Keep HTTP status codes, broker
  acknowledgements, and other transport error mapping in presentation; those
  mappings do not own the application error.
- Remove unused error subclasses. Keep a common base exception only when code
  catches it, adds shared behavior, or intentionally exposes it as a stable
  taxonomy.

### Imports And Wiring

- Import moved contracts from their owning modules in routes, adapters, and
  composition roots. Keep package-level re-exports only when they are an
  intentional, stable public API; otherwise direct imports make ownership and
  dependency wiring explicit.

A typical ownership-oriented refactor has this shape:

```text
application/
├── interactors/create_resource.py   # interactor + create result
├── services/team_validator.py       # validator workflow
└── interfaces.py                    # ports + port DTOs
infrastructure/
├── idempotency_service.py           # database record -> application DTO
└── adapters/user_notifier.py        # adapter-only delivery error
presentation/
└── exception_handlers.py            # application error -> HTTP response
```

Treat colocation as a design convention rather than a layer rule. The hard rule
is that outer representation details must not leak into inner contracts.

## Boundary Checks

- If application logic needs an outer dependency, define a narrow interface in
  the application layer and implement it in infrastructure.
- Do not leak HTTP status codes, broker acknowledgements, request/event models,
  SQLAlchemy rows, dependency injection markers, or external-client types into
  domain/application business rules.
- Settings and runtime framework integration belong outside domain/application
  logic. Inject settings, repositories, gateways, and clients through
  constructors or provider wiring.
- External clients and gateways belong in infrastructure behind application
  interfaces. Use bounded retries only for operations that are safe to retry or
  explicitly idempotent.
- Backward-compatibility hacks for external contracts, such as legacy field
  names, typo-compatible enum values, or deprecated payload shapes, belong in
  presentation or another outer adapter.

## Error Boundaries

- Domain errors describe business rule violations without HTTP language.
- Application errors describe use-case failures and are the normal boundary
  between application logic and presentation error mapping.
- Infrastructure failures should be translated close to the adapter/repository
  into application-owned errors when an application workflow interprets them.
  Otherwise keep implementation-specific errors local to infrastructure.
- Judge ownership by semantic meaning rather than by the first `raise`: an
  outer implementation may detect and raise an inner, application-owned
  outcome while dependencies still point inward.
- Presentation owns transport-specific errors, HTTP status codes, broker
  acknowledgements, and response/event shape.
