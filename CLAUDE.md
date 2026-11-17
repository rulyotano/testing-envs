# Claude Code — Testing Environments

## Project

Go web application for managing dynamic testing environments backed by Docker and Kubernetes. Features auth, per-user CRUD permissions on environments.

## Stack (planned)

- **Language:** Go (latest stable)
- **HTTP:** `net/http` + `chi` router (or bare stdlib if simple enough)
- **Auth:** JWT or session-based
- **Infra:** Docker, Kubernetes
- **DB:** TBD — use repository interfaces so the backing store is swappable

## Architecture

4-layer Clean Architecture. Dependency rule: outer layers depend on inner; inner layers never import outer.

```
cmd/                        → entry points (main.go, wire-up only)
internal/
  domain/                   → entities, value objects, domain errors, repository interfaces
  application/              → use cases, services, DTOs, application interfaces
  infrastructure/           → DB repos, K8s client, Docker client, config, migrations
  presentation/             → HTTP handlers, middleware, router, request/response types
pkg/                        → reusable, import-safe utilities (no internal deps)
```

### Layer responsibilities

| Layer | Owns | Depends on |
|-------|------|------------|
| `domain/` | `Environment`, `User`, `ErrNotFound`, `Repository` interfaces | stdlib only |
| `application/` | `EnvironmentService`, `AuthService`, request/response DTOs | `domain/` |
| `infrastructure/` | `PostgresEnvRepo`, `K8sOrchestrator`, `DockerClient`, config | `domain/`, `application/` |
| `presentation/` | HTTP handlers, middleware, router, JSON marshaling | `application/` |

### Interface placement

- **Repository interfaces** → `domain/` (domain owns what it needs from persistence)
- **Orchestrator / external service interfaces** → `application/` (use cases own what they call)
- **Concrete implementations** → `infrastructure/`

## Go Conventions

### Error Handling

- Wrap errors with context: `fmt.Errorf("creating env %s: %w", name, err)`
- Define sentinel/domain errors in `domain/`: `var ErrEnvNotFound = errors.New("env not found")`
- Callers use `errors.Is` / `errors.As` — never string matching
- Never swallow errors silently; never `panic` in library code

### Interfaces

- Declare interfaces where they are **consumed**, not where implemented
- Keep interfaces small (1–3 methods); compose when needed
- Accept interfaces, return concrete structs

### Naming

- Exported: `EnvironmentService`, `CreateEnvironmentRequest`
- Unexported: `envRepo`, `jwtMiddleware`
- No stutter: `env.Service` not `env.EnvService`
- Error types: `ErrXxx` or `XxxError`

### Context

- First arg always `ctx context.Context` for IO-bound functions
- Propagate; never store in struct
- Respect cancellation in loops and long operations

### Concurrency

- Share memory via channels, not global state
- Always handle goroutine lifecycle (wait groups or errgroup)
- Use `sync.Mutex` only for small critical sections; prefer channels for coordination

### HTTP Handlers

- Handler signature: `func (h *Handler) CreateEnv(w http.ResponseWriter, r *http.Request)`
- Extract request → validate → call service → write response
- Keep handlers thin; business logic lives in `application/`
- Write JSON responses via a shared `respond(w, status, v)` helper
- Return problem details on error: `{"error": "...", "code": "ENV_NOT_FOUND"}`

### Configuration

- Load from env vars + optional config file; use `envconfig` or `viper`
- Fail fast at startup if required config is missing
- Never hard-code credentials; never log secrets

## Testing Practices

### Unit Tests

- Table-driven tests for all logic branches:

```go
tests := []struct {
    name    string
    input   CreateEnvRequest
    wantErr error
}{
    {"valid", validReq, nil},
    {"missing name", noNameReq, ErrInvalidInput},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) { ... })
}
```

- Use `testing` stdlib + `github.com/stretchr/testify` (`assert`, `require`)
- Mock dependencies via interface fakes in `_test` files or a `testutil/` package
- Never test unexported internals directly — test behavior via exported API

### Integration Tests

- Tag with `//go:build integration` and run with `-tags integration`
- Use `testcontainers-go` for real DB/Docker deps in integration tests
- Reset state between tests; never share mutable global state

### Test Helpers

- `testutil/` package for fixtures, fakes, builders
- Builder pattern for test entities:

```go
func NewTestEnv(t *testing.T, opts ...EnvOption) *Environment { ... }
```

### Coverage

- Aim for logic coverage, not line coverage theater
- Table tests cover happy path + each error branch
- HTTP handlers: at least one integration test per endpoint

## Auth & Permissions

- Middleware extracts + validates token; injects user into context
- Use typed context key (`type ctxKey string; const ctxUserKey ctxKey = "user"`)
- Service layer checks permissions; never trust handler-layer claims
- Role model: `admin` can do everything; `user` can CRUD own envs only

## Kubernetes / Docker

- K8s operations go through an interface in `application/` (`Orchestrator`) — swap real client with fake in tests
- Concrete K8s/Docker clients live exclusively in `infrastructure/`
- Never call `kubectl` as subprocess; use `client-go`
- Namespace per testing environment for isolation
- Always set resource limits on created pods/deployments

## Linting & Formatting

- `gofmt` / `goimports` — always
- `golangci-lint` with at least: `errcheck`, `govet`, `staticcheck`, `gosec`
- No unused imports, no unused variables (compiler enforces, also lint)
- Run `go vet ./...` before commit

## What NOT to Do

- No `panic` outside of `main` initialization
- No global mutable state
- No skipping error checks (even `defer f.Close()` — log the error)
- No business logic in HTTP handlers or `presentation/`
- No direct K8s/Docker calls outside `infrastructure/`
- No hardcoded env-specific values (IPs, namespaces, credentials)
- No imports that violate the dependency rule (inner layer importing outer)
