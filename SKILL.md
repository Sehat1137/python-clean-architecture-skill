---
name: python-architecture
description: "Use this skill when working on Python backend architecture for services with HTTP/MQ presentation boundaries: routes or handlers, request/response/event models, application DTOs and errors, interactors/use cases, repository interfaces, clean architecture layer boundaries, dependency inversion, or dependency-injection wiring. Do not use for DB-only migrations, test-only changes, or CI/runtime-only changes unless architecture boundaries are involved."
---

# Python Backend Architecture

## Overview

Use this skill to keep Python backend changes aligned with clean architecture.
Load repository-local instructions first; they define concrete paths, versions,
commands, generated files, and any overrides.

## Workflow

1. Identify the layer being changed and the nearest source of truth in the repo.
2. Apply the Common Closure Principle: identify the owner and reason to change
   for each contract; colocate use-case DTOs and results with the use case, port
   DTOs with the port, and errors in a dedicated module of their owning layer.
3. Keep framework and storage mapping at the outer boundary and business rules
   in domain or application code.
4. Introduce or update interfaces where an outer dependency crosses an inner
   layer boundary.
5. Keep dependency injection consistent with the repo's existing container and
   provider pattern.
6. Validate with the narrowest tests/checks that cover the changed behavior.

Read focused references only when needed:

- Read `references/clean-architecture.md` for layer-boundary, contract or error
  ownership, DTO placement, or dependency-direction decisions.
- Read `references/interactors.md` when adding or changing use cases, DTOs, or
  application orchestration.
- Read `references/presentation-boundaries.md` for HTTP routes, MQ/event
  handlers, request/response/event models, OpenAPI/AsyncAPI, transport
  semantics, or authorization-boundary work.
- Read `references/dependency-injection.md` when changing provider wiring or
  injected dependencies.
- Read `references/evals.md` when reviewing skill activation, output quality, or
  whether this skill should apply to an ambiguous request.

## Hard Rules vs Conventions

Treat dependency direction, layer isolation, transport/persistence boundaries,
and keeping outer representations and transport error mapping out of inner
contracts as hard rules. Treat colocating contracts by owner and reason to
change, naming, shared module shape, and dependency-injection organization as
conventions unless the repository defines a stronger local pattern.

## Layer Guidance

- Domain code owns entities, value objects, and domain rules. Avoid importing
  web/MQ frameworks, database clients, dependency-injection markers, external
  clients, or transport schemas into domain code.
- Application code owns use cases, DTOs, ports/interfaces, orchestration, and
  application-level errors. Keep it independent from concrete infrastructure.
- Infrastructure code owns database tables, repositories, external clients, and
  adapters. It may implement application interfaces but should not define the
  business contract.
- Presentation code owns HTTP routes, MQ/event handlers, transport
  request/response/event models, OpenAPI/AsyncAPI, dependency-injection
  integration, and transport error translation.

## API and Application Boundaries

- Keep routes and handlers thin: parse and validate input, call an application
  interactor or service, then map the result to a transport response,
  acknowledgement, or published event.
- Presentation may import application interactors and DTOs directly. Do not add
  interfaces between presentation and application only for dependency inversion.
- Prefer explicit request, response, and event models at transport boundaries.
  Use DTOs or typed application inputs between presentation and application
  layers when the transport schema is not the business contract.
- Add one focused interactor/use-case object when a workflow has meaningful
  orchestration, persistence, authorization, or external-service coordination.
- Return domain or application result objects from interactors; avoid returning
  database rows or transport models from application code.
- Keep HTTP status codes, broker acknowledgements, OpenAPI/AsyncAPI behavior,
  and transport-specific errors in presentation code.
- Keep backward-compatibility behavior for external contracts, such as legacy
  field names or deprecated payload shapes, in presentation adapters.

## Repositories and Dependency Injection

- Define repository or gateway interfaces in the inner layer where the contract
  is consumed. Implement them in infrastructure.
- Keep transport payload and database-record mapping in presentation or
  infrastructure. Keep application DTOs independent of Pydantic models,
  database rows, column names, and constructors such as `from_record`.
- Keep repository interfaces focused on aggregates or use cases, not generic
  database access.
- Bind implementations through the repo's existing dependency-injection system.
  Do not introduce global singletons for runtime dependencies.
- Preserve existing provider naming, scopes, and request lifecycle conventions.

## Validation

- For presentation or transport behavior changes, add or update presentation
  tests and regenerate OpenAPI/AsyncAPI when the repo requires it.
- For application orchestration changes, add focused use-case tests.
- For repository contract changes, add persistence tests or mocks at the layer
  boundary, depending on the repo's existing pattern.
- For ownership-only refactors with unchanged behavior, update existing imports
  and contract tests; do not add tests solely for a class's file location.
- Report any migration, OpenAPI/AsyncAPI, DI, or public contract impact
  explicitly.
