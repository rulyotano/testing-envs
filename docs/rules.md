# Development Rules

Conventions for this project. All contributors (human and AI) follow these.

## Architecture — 4-Layer Clean Architecture

```
internal/
  domain/          core: entities, value objects, errors, repository interfaces
  application/     use cases: services, DTOs, orchestrator interfaces
  infrastructure/  external: DB, K8s, Docker, config, migrations
  presentation/    delivery: HTTP handlers, middleware, router, JSON types
```

### Dependency rule

```
presentation  →  application  →  domain
                 infrastructure →  domain + application
cmd           →  everything (wire-up only)
```

No inner layer imports an outer layer. Violations are architecture bugs.

### Interface placement

| Interface type | Lives in |
|----------------|----------|
| `EnvironmentRepository` | `domain/` — domain owns its persistence contract |
| `UserRepository` | `domain/` |
| `Orchestrator` (K8s/Docker) | `application/` — use cases own what they orchestrate |
| Concrete implementations | `infrastructure/` |

## Go Idioms

### Errors

```go
// Define in domain/
var ErrEnvironmentNotFound = errors.New("environment not found")

// Wrap with context everywhere
return nil, fmt.Errorf("get environment %s: %w", id, ErrEnvironmentNotFound)

// Check by callers — never string-match
if errors.Is(err, domain.ErrEnvironmentNotFound) { ... }
```

### Interfaces

```go
// In domain/ — small, focused
type EnvironmentRepository interface {
    Get(ctx context.Context, id string) (*Environment, error)
    List(ctx context.Context, ownerID string) ([]*Environment, error)
    Save(ctx context.Context, env *Environment) error
    Delete(ctx context.Context, id string) error
}

// In application/ — orchestration contract
type Orchestrator interface {
    ProvisionEnvironment(ctx context.Context, env *domain.Environment) error
    TeardownEnvironment(ctx context.Context, envID string) error
}
```

### Context

```go
// Always first argument
func (s *EnvironmentService) Create(ctx context.Context, req CreateRequest) (*domain.Environment, error)

// Typed key — never plain string
type ctxKey string
const ctxUserKey ctxKey = "user"

func UserFromContext(ctx context.Context) (*domain.User, bool) {
    u, ok := ctx.Value(ctxUserKey).(*domain.User)
    return u, ok
}
```

## HTTP (presentation layer)

### Response helpers

All handlers use shared helpers — no raw `json.NewEncoder` calls in handlers:

```go
respond(w, http.StatusOK, payload)
respondError(w, http.StatusNotFound, "environment not found")
```

### Error response shape

```json
{
  "error": "human readable message",
  "code": "ENV_NOT_FOUND"
}
```

### Domain error → HTTP status mapping

Mapping lives in `presentation/` — `application/` and `domain/` know nothing about HTTP:

```
ErrNotFound       → 404
ErrUnauthorized   → 403
ErrForbidden      → 403
ErrInvalidInput   → 400
ErrConflict       → 409
all others        → 500
```

### Middleware order

```
Logger → Recovery → Auth → CORS → Router
```

Auth middleware injects user into context. All authenticated routes assume user is present.

## Testing

### Unit tests

- File: `foo_test.go` next to `foo.go`
- Package: `package foo_test` (black-box) unless testing unexported helpers
- Always table-driven
- Use `testify/assert` (non-fatal) and `testify/require` (fatal, for setup/preconditions)
- Fakes over mocks: implement the interface manually in `testutil/`

```go
// testutil/fake_env_repo.go
type FakeEnvironmentRepository struct {
    envs map[string]*domain.Environment
}

func (f *FakeEnvironmentRepository) Get(ctx context.Context, id string) (*domain.Environment, error) {
    env, ok := f.envs[id]
    if !ok {
        return nil, domain.ErrEnvironmentNotFound
    }
    return env, nil
}
```

### Integration tests

- Build tag: `//go:build integration`
- Run: `go test -tags integration ./...`
- Use `testcontainers-go` for real infrastructure (Postgres, etc.)
- Each test is self-contained: setup → act → assert → cleanup

### Benchmarks

- File: `foo_bench_test.go`
- Name: `BenchmarkFoo`
- Run: `go test -bench=. -benchmem ./...`

## Security

- Validate all external input at the `presentation/` boundary
- Never log tokens, passwords, or secrets
- Use `gosec` linter; fix all HIGH severity findings
- Env vars for all credentials; no secrets in committed files
- Namespace K8s resources per environment for blast-radius isolation

## Kubernetes / Docker

- All K8s operations through `application.Orchestrator` interface
- Concrete implementation in `infrastructure/k8s/`
- Use `client-go`, never shell out to `kubectl`
- Set CPU + memory limits on every workload created
- Label all created resources: `app=testing-envs`, `owner=<user-id>`
- Deleting an environment must delete its entire namespace

## Commits & PRs

- Conventional Commits: `feat:`, `fix:`, `chore:`, `test:`, `refactor:`
- Each PR: one concern, reviewable in < 400 lines diff
- PR description: what changed and why (not how — the code shows how)
- All tests green before merge; no linter suppressions without comment

## Linting

Required checks (CI and local):

```bash
gofmt -l .              # no output = clean
go vet ./...
golangci-lint run
```

Minimum enabled linters: `errcheck`, `govet`, `staticcheck`, `gosec`, `goimports`, `misspell`
