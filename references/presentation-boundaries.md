# Presentation Boundaries Reference

## Handler Design

- Prefer separate routes, subscribers, or handlers for distinct business
  operations instead of overloading one transport entrypoint with unrelated
  modes.
- Keep handler functions thin: parse transport inputs, build DTOs/auth context,
  call interactors/services, and map results to response models,
  acknowledgements, or published events.
- Organize routes and handlers by feature or aggregate following nearby modules.
- For HTTP, use path parameters for resource identity, including `{path:path}`
  only when a file-like path can contain slashes.
- For HTTP, use query parameters for filtering, sorting, pagination, and
  optional selectors.
- For MQ/events, keep event filtering, envelope parsing, and acknowledgement
  behavior in presentation; keep business decisions in application interactors.

## Models and Validation

- Use separate request, response, and event models when the public transport
  shape differs from domain/application objects.
- Response/event output models should include only fields needed by clients or
  downstream consumers.
- Transport validation belongs in Pydantic/framework models when it is syntactic
  or contract-level; business validation belongs in application/domain code.
- If a transport field becomes optional or nullable, update transport models,
  DTOs/domain types, persistence, and tests together.
- Use framework boundary constraints, such as Pydantic `Field`, FastAPI
  `Query`/`Path`, or message schema validation, for boundary validation and
  documentation.
- Map request/event models to application DTOs explicitly. Do not pass transport
  models deep into application logic.
- Keep OpenAPI/AsyncAPI descriptions accurate when changing transport shapes.
- Prefer Pydantic v2-style validators and model configuration when custom
  validation is required.

## HTTP Semantics

- `200 OK`: successful reads and updates with response bodies.
- `201 Created`: successful create operations.
- `204 No Content`: successful delete or state-change operations with no body.
- `400 Bad Request`: malformed client input not handled by model validation.
- `403 Forbidden`: authenticated user lacks permission.
- `404 Not Found`: requested resource does not exist or is not visible.
- `409 Conflict`: uniqueness, collision, or business-state conflicts.
- `422 Unprocessable Entity`: semantic validation failures.

## MQ/Event Semantics

- Treat incoming events as external transport contracts. Parse and validate the
  envelope in presentation before building application DTOs.
- Keep idempotency keys, event timestamps, action filters, retry/DLQ behavior,
  and acknowledgement rules explicit.
- Application interactors should receive typed DTOs, primitives, domain objects,
  or auth/context objects, not raw message envelopes.
- When event handling updates state, test outdated-event, duplicate-event,
  invalid-payload, and retry/DLQ-relevant paths where the repository supports
  them.

## Authorization Boundary

- Use the repository's trusted auth context model instead of accepting identity,
  roles, or teams from request bodies or query parameters.
- Authorization decisions that depend on business state belong in application
  services/interactors, not ad hoc route checks.
- Public routes/topics should be explicitly separated from private
  routes/topics and should not accidentally expose private-only data.
- When permissions change, test both allowed and denied cases at presentation or
  application level.

## Error Mapping

- Import domain and application errors from their owning modules and map them to
  transport responses or acknowledgements at the presentation boundary.
- Do not move or group application errors by HTTP status, broker outcome, or
  handler function. Transport mapping is a separate reason to change.
- Preserve typed error identity when several handlers share the same transport
  response; do not replace semantically distinct errors with a generic
  transport-oriented application exception.

## Presentation Tests

- Assert status code or acknowledgement behavior, response/event shape, important
  field values, and error mapping.
- For validation changes, test the boundary that owns the rule.
- For contract changes, update request/response/event models,
  OpenAPI/AsyncAPI-facing descriptions, and tests together.
