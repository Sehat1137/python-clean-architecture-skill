# Interactors Reference

## Shape

- Use one interactor/use-case object per distinct business workflow when the
  service uses interactors.
- Name interactors after the use case, usually verb-noun with an `Interactor`
  suffix, unless the repo has a stronger local convention.
- Prefer one public entrypoint (`__call__`, `execute`, or the local convention)
  and keep its signature focused.
- Accept application DTOs, domain objects, primitive identifiers, or auth
  context objects. Do not accept transport request/event models.
- Return domain entities, application DTOs, or explicit result objects. Do not
  return database records or response models.
- Keep an input or result type used by only one use case next to that interactor
  so the workflow and its contract change together. Move it to a shared feature
  module only when multiple use cases genuinely own the contract.
- Avoid adding single-field DTOs or extracted private helpers by default. Add
  them when they represent a real layer boundary, remove meaningful duplication,
  clarify a non-trivial workflow, or match a strong local pattern.
- Split interactors when one class starts serving multiple endpoints, queue
  handlers, or workflows for different reasons. A reusable application service is
  acceptable only when it owns genuinely shared behavior and does not branch by
  caller.

## Dependencies

- Inject repositories through application interfaces, not infrastructure
  implementations.
- Inject clocks, ID generators, permission services, validators, and gateways
  when they are part of the workflow.
- When adding a dependency, update the interface when needed, provider wiring,
  tests/fakes, and public exports if the repo uses exports.
- If an interactor needs many collaborators, check whether reusable application
  service logic should be extracted.

## Workflow and Transactions

- Keep persistence details in repositories and transport details in presentation.
- Validate business preconditions before persistence and let
  domain/application errors surface through existing handlers.
- Use transactions for multi-step writes that must succeed or fail together.
- Keep transaction scope minimal. Do validation before opening or committing
  expensive work when practical.
- Do not duplicate authorization or validation ad hoc in presentation when the
  decision belongs to the use case.

## Testing

- Unit-test orchestration with fakes or typed mocks for application interfaces.
- Cover success, important failure paths, and "must not call dependency"
  behavior for rejected operations.
- Avoid tests for private transaction mechanics unless they are observable
  through state, calls, or returned errors.
