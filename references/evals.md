# Architecture Skill Evals

Use these lightweight checks when reviewing activation or output quality.

## Should Trigger

- "Add an HTTP endpoint and wire it to a use case."
- "Add a Kafka/FastStream message handler and map the event to an application DTO."
- "Review whether this DTO belongs in application or presentation."
- "Review whether this one-field DTO or private interactor helper is justified."
- "Refactor a generic application/dto.py so contracts live with their owning use cases or ports."
- "Move asyncpg record mapping out of an application DTO without changing the port contract."
- "Review application error ownership according to the Common Closure Principle."
- "Decide whether a gateway failure belongs beside the adapter, port, use case, or HTTP handler."
- "An infrastructure idempotency service detects a conflict. Which layer owns the typed error?"
- "A repository returns no record. Is ResourceNotFoundError an application or domain error?"
- "Refactor repository dependencies behind Protocol interfaces."
- "Check dependency-injection wiring for a new interactor."

## Should Not Trigger

- "Only add an Alembic migration for a new index."
- "Only update pytest parametrization in an existing test."
- "Only adjust Docker, CI, or uvicorn startup."
- "Only rename an exception module and update imports without changing ownership or boundaries."

## Output Quality Checks

- Identifies touched layers and the source-of-truth local conventions.
- Keeps HTTP/MQ framework, Pydantic transport models, SQLAlchemy, DI framework,
  and external-client details out of inner business modules.
- States whether a rule is a hard boundary or a local convention.
- Maps transport schemas to application DTOs explicitly instead of passing raw
  request/event models into interactors.
- Keeps resource-existence, idempotency, and version/state decisions inside the
  application layer while allowing presentation to map typed outcomes to
  transport statuses.
- Avoids unnecessary single-field DTOs or private helper extraction unless they
  carry a real boundary, reuse, or complexity reduction.
- Uses contract ownership and the Common Closure Principle to colocate
  use-case-specific types with interactors and port-specific types with
  interfaces instead of defaulting to a generic DTO module.
- Keeps database rows, external payload parsing, and transport mapping in outer
  adapters while preserving plain application-owned contracts.
- Places errors in dedicated modules of the layer or feature that gives them
  semantic meaning instead of putting exception classes in interface modules.
- Assigns error ownership to the workflow or layer that gives a failure
  semantic meaning, not automatically to the component that detects or raises
  it.
- Allows an infrastructure implementation to raise an application-owned typed
  outcome, such as an idempotency conflict, because the dependency still points
  inward.
- Treats failure to load a requested entity as an application outcome by
  default, while keeping intrinsic entity and value-object invariant violations
  in domain.
- Keeps adapter-only errors in infrastructure when application code does not
  interpret them.
- Keeps HTTP/MQ error mapping in presentation instead of organizing application
  errors by transport status.
- Removes dead errors and avoids common base exceptions that provide no shared
  handling, behavior, or intentional taxonomy.
- For Protocol/interface or DI changes, includes enough concrete wiring detail
  to show where the interface, implementation, provider/container, and tests
  change.
- Updates direct imports, intentional package exports, and DI registrations
  after moving contract types.
- Mentions API, OpenAPI/AsyncAPI, DI, migration, or public-contract impact when
  relevant.
- Names targeted validation and any skipped checks.
