# Dependency Injection Reference

- Use the repository's existing dependency-injection framework and provider
  organization; do not introduce hidden global construction for runtime
  dependencies.
- Keep providers separated by layer where existing patterns do.
- Prefer request scope for request-bound services, session scope for database
  sessions/connections, and singleton scope only for stateless configuration or
  shared infrastructure objects.
- Register interfaces to implementations explicitly when the DI framework
  supports it.
- When constructor dependencies change, check provider wiring, app factory
  setup, endpoint injection, and tests together.

## Provider Organization

- Application providers should focus on use cases, services, validators, and
  application-level dependencies.
- Infrastructure providers should focus on database sessions, repositories,
  external clients, settings-backed adapters, and concrete implementations.
- Container creation and shutdown belong in app setup/lifespan code, not inside
  route handlers.

## Wiring Pattern

When adding a new interactor or interface-backed repository, update wiring in
the same change as the constructor signature. A useful review path is:

- the application interface exists before the infrastructure implementation;
- the implementation is registered against that interface or local provider
  contract;
- the route, handler, or subscriber asks the container for the interactor or
  service, not for infrastructure details;
- app factory, container factory, lifespan setup, and tests still use the same
  scope and lifecycle conventions as nearby code.

For repositories that use Dishka, the common local shape is to register
interactors in application with `provide_all`, and bind infrastructure
implementations to application interfaces with `provide(..., provides=...)`:

```python
# application/ioc.py
from dishka import Provider, Scope, provide_all

from app.application.interactors.create_tenant import CreateTenantInteractor
from app.application.interactors.update_tenant import UpdateTenantInteractor


class ApplicationProvider(Provider):
    interactors = provide_all(
        CreateTenantInteractor,
        UpdateTenantInteractor,
        scope=Scope.REQUEST,
    )
```

```python
# infrastructure/ioc.py
from dishka import Provider, Scope, provide

from app.application import interfaces
from app.infrastructure.repository.tenant import TenantRepository


class InfrastructureProvider(Provider):
    tenant_repository = provide(
        TenantRepository,
        scope=Scope.REQUEST,
        provides=interfaces.TenantRepository,
    )
```

Use explicit `@provide` factory methods when a dependency needs construction
logic, derived settings, or several collaborators instead of direct class
registration:

```python
# application/ioc.py
from dishka import Provider, Scope, provide

from app.application.services.auth_settings import AuthSettingsService
from app.application.services.endpoint_settings import EndpointSettings
from app.config import Config


class ApplicationProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def auth_settings_service(
        self,
        config: Config,
        endpoint_settings: list[EndpointSettings],
    ) -> AuthSettingsService:
        return AuthSettingsService(
            scopes=config.authorization.scopes,
            skip_authentication=not config.authentication.enabled,
            skip_authorization=not config.authorization.enabled,
            teams=config.authorization.teams,
            endpoint_settings=endpoint_settings,
        )
```

Factory providers are also useful for async lifecycle, context values, or test
overrides, such as database connections, settings-backed clients, clocks, and ID
generators. If the repository uses constructor injection, provider classes,
decorators, FastAPI dependencies, or a different container API, preserve that
local pattern and apply the same boundary: wire outer implementations to inner
interfaces at the composition root.

## Testing

- In tests, override dependencies through test providers/containers or construct
  interactors directly with fakes/mocks.
- Preserve request/session isolation for dependencies that hold per-request
  state.
- If provider wiring changes, add at least one test path that exercises the
  injection boundary when practical.
