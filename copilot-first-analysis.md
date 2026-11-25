I’ll compare the architecture proposal against the repository’s current rules and README, then highlight gaps and concrete alternatives.

Read 
Completed with input: {
  "path": "/Users/ruly/projects/personal/testing-envs/ARCHITECTURE.md"
}

Read 
Completed with input: {
  "path": "/Users/ruly/projects/personal/testing-envs/README.md"
}

Read 
Completed with input: {
  "path": "/Users/ruly/projects/personal/testing-envs/docs/rules.md"
}

Read 
Completed with input: {
  "path": "/Users/ruly/projects/personal/testing-envs/AGENTS.md"
}

Search Tools

The key issue is not component naming; it is defining the provisioning lifecycle, ownership boundaries, and failure/retry behavior before choosing technologies.

Search repositories
Completed with input: {
  "query": "ephemeral preview environments Kubernetes ArgoCD topic:preview-environments",
  "minimal_output": true,
  "perPage": 5
}

Read skill `show-github-search-result`
Completed with input: {"skill":"show-github-search-result"}

I reviewed `ARCHITECTURE.md`, `AGENTS.md`, `docs/rules.md`, and `README.md`.

## Overall assessment

The proposal is a reasonable POC architecture, and it provides a clear starting point:

- Docker, Kubernetes, and Argo CD are suitable foundations.
- Clean Architecture gives the API good testability.
- A namespace per environment is a good isolation boundary.
- The separation between API and task processor is useful for asynchronous work.
- It intentionally prioritizes delivery over excessive abstraction.

The main weakness is that it describes components better than behavior. Several important flows—retry, partial failure, rollback, ownership, state changes, and deletion—remain underspecified.

## Important findings

### 1. Add an explicit provisioning lifecycle

A single `Created` status is insufficient.

I would use an environment state machine such as:

```text
Pending
  -> Validating
  -> BuildingImages
  -> ProvisioningResources
  -> WaitingForDeployment
  -> Ready
  -> Deleting
  -> Deleted

Any state -> Failed
```

Track each service and operation independently. Also persist:

- Requested branch
- Resolved commit SHA
- Immutable image tag
- Argo CD application name
- Created resources
- Error and recovery state
- Expiration timestamp
- Started and completed timestamps

A service based on `latest` branch state is not reproducible: a rebuild or redeployment could silently use a newer commit.

### 2. Treat image builds as separate pipeline jobs

A worker should not synchronously build images or continuously poll with a cron job.

Prefer:

- API publishes an image-build request.
- CI or a dedicated build worker builds `commitSHA`-tagged images.
- Registry or build-pipeline events publish success or failure.
- The provisioning worker resumes only after successful builds.
- Failed builds support retry, backoff, and cancellation.

The immutable key should be something like:

```text
registry/service:<commit-sha>
```

### 3. RabbitMQ should not be the state machine

RabbitMQ is useful for commands and jobs, but not as the sole source of truth.

Every environment or provisioning operation should have:

- A durable database record
- A unique operation ID
- A monotonically tracked status
- Idempotent workers
- Correlation IDs across messages
- Outbox or transaction-based event publishing

Also, “dynamic queues per environment” will quickly become operationally expensive. A small number of durable queues with routing or message-type fields is usually sufficient.

For a POC, it may even be better to start with a database-backed worker rather than RabbitMQ. Introduce the broker once retry behavior or asynchronous scale justifies it.

### 4. Clarify Terraform versus Argo CD responsibilities

The proposal conflates two control planes:

- Argo CD deploys Kubernetes workloads declaratively.
- Terraform or Crossplane provisions external infrastructure and Kubernetes resources.

If Argo CD deploys Terraform files, add a Terraform runner or separate operator. Otherwise, the current process is unclear.

I would prefer:

- Helm or Kustomize for application manifests.
- Argo CD Application or ApplicationSet for deployment.
- Terraform or Crossplane for external databases, DNS, certificates, or cloud resources.
- One desired-state record per environment.

### 5. Do not copy and mutate a shared GitOps repository per environment

Writing generated files into `argo-cd-devops` creates repository churn and concurrent-write conflicts.

A better pattern is:

- Templates are stored as Helm/Kustomize charts.
- Environment parameters are stored in the application database.
- A reconciler creates or updates an Argo CD `Application`.
- Kustomize or Helm generates the rendered resources.
- Argo CD remains the deployment state reconciler.

ApplicationSet can scale this further for labels and cluster selection.

### 6. Add deletion and rollback semantics

Deleting an environment must be a tracked operation.

At minimum it should remove:

- Argo CD applications
- Kubernetes namespace
- Databases and seeds
- DNS records
- Build jobs
- Temporary secrets
- Metrics and cost-allocation metadata

A TTL and expiration policy are especially important for preventing permanent resource growth.

### 7. Define the database model and database responsibility

MongoDB may work for a POC, but it does not eliminate the need for clear repositories and migration strategy.

I would separate:

- `environments`
- `environment_services`
- `provisioning_operations`
- `service_definitions`
- `image_builds`
- `users`
- `audit_events`

Operational state does not need to live in the same database that stores service data. MongoDB plus PostgreSQL without a clear reason for each will add complexity.

### 8. Security is underdefined

The architecture needs additional requirements for:

- JWT or session revocation
- Admin/user/owner authorization checks
- Namespace isolation
- Kubernetes RBAC
- Network policies
- Secrets management
- Signed and scanned images
- Preserving image provenance
- Resource requests and limits
- Resource quotas
- CPU, memory, database, and cost limits
- Audit logging
- TLS and wildcard certificate automation
- DNS name reservation and collision protection
- No secrets in generated Terraform or Helm values
- Branch authorization for private repositories

### 9. Define API behavior

Important endpoints and fields are missing.

A useful contract is:

```text
POST   /environments
GET    /environments
GET    /environments/{id}
PATCH  /environments/{id}
DELETE /environments/{id}
GET    /environments/{id}/events
GET    /environments/{id}/logs
GET    /services
POST   /admin/services
GET    /branches?repository=...
```

Responses should expose status, commit, services, endpoint, owner, timestamps, and errors—not only a human-readable status.

## Similar solutions and patterns

| Solution | Architectural similarity |
|---|---|
| Argo CD ApplicationSet | GitOps-based generation of one application per environment |
| Crossplane | Declarative control plane for Kubernetes and cloud resources |
| Okteto Preview Environments | Branch-based isolated Kubernetes environments |
| Uffizzi | Ephemeral Docker/Kubernetes environments |
| Qovery | Internal developer platform for environment provisioning |
| Bunnyshell | Ephemeral environments and developer workflows |
| Vercel/Netlify/Heroku review apps | Preview environment lifecycle, routing, and cleanup |

A close GitHub example found through repository search is [nitishhansa1/preview-env-platform](https://github.com/nitishhansa1/preview-env-platform), described as ephemeral per-PR environments using GitHub Actions, Argo CD, and Helm. It is a small example rather than production-validated software, but it illustrates common patterns: CI-triggered provisioning and GitOps deployment.

## What I would change in the instructions

I would replace the broad component list with these required sections:

1. **Control-plane architecture**
   - API
   - Worker/reconciler
   - GitOps reconciler
   - CI/build integration
   - Registry
2. **Environment aggregate**
   - Ownership
   - State
   - Services
   - Revisions
   - Resources
   - TTL
3. **State machine**
   - All transitions
   - Terminal states
   - Retry behavior
   - Idempotency
4. **API contract**
   - Routes
   - Status codes
   - Error format
   - Pagination
5. **Infrastructure contract**
   - Namespace layout
   - Labels
   - Resource limits
   - Network policies
   - Cleanup
6. **Asynchronous communication**
   - Message formats
   - Delivery semantics
   - Event ordering
   - Dead-letter handling
7. **Security and tenancy**
   - Roles
   - Isolation
   - Secrets
   - Audit
8. **Observability**
   - Structured logs
   - Correlation IDs
   - Metrics
   - Provisioning traces
9. **Recovery**
   - Orphan cleanup
   - Partial deployment
   - Reconciliation
   - Manual retry
10. **Cost and lifecycle**
    - TTL
    - Idle shutdown
    - Quotas
    - Expiration

## Recommended implementation sequence

For the POC, I would start with:

1. API plus PostgreSQL control-plane state.
2. Environment and service state machines.
3. Helm/Kustomize templates.
4. Argo CD Application creation.
5. CI or registry image resolution.
6. A small database-backed worker.
7. Namespace-level deletion and TTL cleanup.
8. RabbitMQ only after the async flow is validated.
9. MongoDB only for a concrete service-backed requirement, not as a default extra component.

The current architecture is a valid first draft, but it should be a reconciliation system rather than a sequence of imperative handler steps.