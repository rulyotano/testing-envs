# AGENTS — Testing Environments

Instructions for AI agents (Claude Code, Cursor, Copilot, Codex, etc.) working in this repository.

## Project Summary

Go web application to create and manage dynamic testing environments backed by Docker and Kubernetes. Users authenticate, then create/view/update/delete environments based on role permissions.

## Repository Layout

```
cmd/                        entry points (wire-up only)
internal/
  domain/                   entities, value objects, domain errors, repository interfaces
  application/              use-case services, DTOs, orchestrator interfaces
  infrastructure/           DB repos, K8s client, Docker client, config, migrations
  presentation/             HTTP handlers, middleware, router, JSON types
pkg/                        shared utilities (no internal deps)
```

### Dependency rule

```
presentation → application → domain
infrastructure → domain, application
cmd → everything
```

Inner layers never import outer. Compiler enforces this via `internal/`.

## Decision Rules for Agents

| Situation | Where the code goes |
|-----------|---------------------|
| New entity or domain error | `domain/` |
| New use case | `application/` service |
| New external interface (repo, orchestrator) | Interface in `domain/` or `application/`; impl in `infrastructure/` |
| New HTTP endpoint | Handler in `presentation/`; thin, delegates to `application/` |
| New DB query | `infrastructure/` repository implementation |
| New K8s / Docker op | `infrastructure/`; called via interface from `application/` |
| Auth / permission change | Middleware in `presentation/` + permission check in `application/` service |

## Code Style (Go)

- `gofmt` / `goimports` always applied
- Errors: wrap with `%w`, use `errors.Is`/`errors.As`, define sentinels in `domain/`
- Interfaces: declared at point of consumption (small, 1–3 methods)
- Context: first arg on every IO function, propagate, never store in struct
- No global mutable state
- No business logic in `presentation/`

## Testing

- Every new function gets a table-driven unit test
- Fakes implement the interface manually; live in `testutil/` or `_test.go`
- Integration tests: build tag `//go:build integration`
- Run before finishing any task:
  ```
  go test ./...
  go vet ./...
  golangci-lint run
  ```

## Permissions Model

- `admin`: full CRUD on all environments
- `user`: CRUD on own environments only
- Permission checks live in `application/` service, not in `presentation/`

## Never Do

- Call K8s / Docker APIs outside `infrastructure/`
- Store secrets in code or config files committed to git
- Skip error checks
- Add business logic to HTTP handlers
- Break the dependency rule (inner layer importing outer)
- Use `panic` outside `main` init

## Definition of Done

1. Feature works end-to-end
2. Unit tests pass for new logic
3. `go vet ./...` clean
4. `golangci-lint run` clean
5. No new linter suppressions without explaining comment
