I read `ARCHITECTURE.md` (the IDE path says `architecture.md`, but the file on disk is uppercase), and checked it against `CLAUDE.md`. Overall it's a solid start: the flow makes sense and keeping per-service config in each repo is right. But there's one wrong assumption about Argo CD, and several things missing that will hurt once the system is running.

## Biggest problems

**1. Argo CD doesn't run Terraform.** The doc says to "copy terraform files into argo-cd directory". Argo CD applies Kubernetes manifests (Helm, Kustomize, plain YAML). It won't run `.tf` files.
- Fix: each service ships a Helm chart or Kustomize overlay in `.devops/testing/`, and the environment name is passed in as a value.
- Keep Terraform only for long-lived infrastructure (the cluster, DNS, the registry).

**2. `CLAUDE.md` and `ARCHITECTURE.md` disagree.**
- `CLAUDE.md` says the platform talks to Kubernetes directly with `client-go`, using an `Orchestrator`. `ARCHITECTURE.md` is GitOps: the platform writes to git and Argo does the deploying. Pick one. With GitOps, the "Orchestrator" just writes to git, and `client-go` is only needed to read Argo's status.
- `CLAUDE.md` mentions a `DockerClient`. Image builds should happen in CI, not in the API.
- `CLAUDE.md` uses Postgres repositories as its example, while `ARCHITECTURE.md` picks MongoDB for the API.
- `CLAUDE.md`, `AGENTS.md` and `.cursorrules` repeat the same rules, so they'll drift apart. Keep one source and link to it from the others.

**3. Status is underspecified.** Right now it's "Created, then wait for Argo", and there's no failure state. You need a clear state machine:
```
Pending → BuildingImages → Deploying → Ready
                ↓              ↓
              Failed ←────────┘
Ready → Deleting → Deleted
```
Each service needs its own status as well. For "wait for Argo", read each Argo `Application`'s health and sync status (through Argo notifications/webhooks, or by watching the resource). For builds, have the CI pipeline call back to a webhook, and keep a slower scheduled check as a fallback. Polling by cron alone is too slow.

**4. RabbitMQ + task processor vs. a reconcile loop.** An event-driven design needs retries, idempotency, a dead-letter queue and handling for out-of-order messages, and none of that is in the doc yet. The alternative is the controller pattern: the database holds the desired state, and a loop keeps comparing it to the actual state and fixing differences. Crashes recover on their own. You could even make an environment a Kubernetes custom resource and write a small operator. For the POC, a reconcile loop is simpler than a queue.

## Missing things

| Area | Gap |
|------|-----|
| **Cleanup / TTL** | Environments get forgotten. Add an expiry date, auto-delete, and per-user limits. This is the main cost risk. |
| **Delete flow** | Only "delete testing env" is mentioned. A full delete has to remove the git files, the namespace, the databases, the queues, DNS/TLS, and old images. |
| **Name rules** | "Letters and numbers" isn't enough. Names must be valid DNS labels: lowercase, 63 characters max, starting with a letter. Leave room for prefixes like `env-<name>-db`. Add a unique index so two users can't create the same name at once. |
| **Image tags** | Tag images with the commit SHA. The registry then acts as the cache (does that tag exist?), so the "commit-id:image-tag" map in the database isn't needed. |
| **Branch updates** | Not decided: is an environment pinned to one commit, or does it redeploy when someone pushes to the branch? Add a "redeploy / update" action. |
| **Services not selected** | If an environment deploys services A and B, and B calls C, where does C come from? Either fall back to a shared baseline environment, or have services declare their dependencies. |
| **Databases** | Create and seed them inside the cluster (a Kubernetes Job, Helm hook or init container), not from the API. A small database inside each environment's namespace is easier to clean up than a shared database server. |
| **RabbitMQ roles** | "Dynamic queues for envs" mixes two things: the platform's own messaging and the queues used by the services being tested. For the services, use one RabbitMQ vhost per environment, or a broker inside its namespace. |
| **Isolation** | Add a ResourceQuota, LimitRange and NetworkPolicy to each namespace. |
| **Secrets** | Not mentioned. Use External Secrets or Sealed Secrets, and never put secrets in the GitOps repo. |
| **Ingress / TLS** | Wildcard DNS and a wildcard certificate for `*.testings.company-url.com` (cert-manager), plus an ingress controller. |
| **Auth** | Use single sign-on through your existing identity provider (OIDC via GitLab, Keycloak or Dex) instead of a separate user store. |
| **Observability** | The details page should link to logs, pod status and Argo for that environment. |
| **Config schema** | Define a versioned schema for each repo's `.devops/testing-env.yaml`, so the platform can validate it. |

## What I'd change

1. Replace "terraform templates" with Helm or Kustomize everywhere.
2. Use **one** database for the platform for the POC. Mongo and Postgres together is extra work. Postgres matches `CLAUDE.md`.
3. Drop RabbitMQ from the platform for the POC and use a reconcile loop. Keep RabbitMQ only as a dependency of the services being tested.
4. Look hard at Argo CD's **ApplicationSet** feature. Its git file generator can create one environment per config file, so the platform only has to commit `envs/<name>/config.yaml` and Argo does most of the rest. That could remove much of the task processor.
5. Run everything locally with k3d or kind for the POC.
6. Fix typos: "under the hook" → "under the hood", "RabbitQ" → "RabbitMQ", "fullfil" → "fulfill".

## Similar solutions

- **Argo CD ApplicationSet (Pull Request generator):** preview environments per PR, built into Argo. Closest to your plan.
- **Signadot:** deploys only the changed services and routes requests by header to a shared baseline for everything else. Much cheaper than a full stack per environment, and it directly addresses the "services not selected" problem.
- **mirrord / Telepresence:** connect a local process into a shared cluster, a lighter alternative.
- **Okteto, Bunnyshell, Qovery, Release, Shipyard, Northflank:** commercial "environments as a service" products. Compare features before building your own.
- **GitLab Review Apps:** per-branch environments built into GitLab CI. A good fit if you choose GitLab.
- **vCluster:** a virtual cluster per environment, giving stronger isolation than a namespace.
- **Backstage:** could host the UI as a plugin instead of a separate Vue app.

Before building, decide whether each environment is a **full copy** of the stack (your current plan: simple, expensive) or a **baseline plus only the changed services** (the Signadot model: complex routing, cheap). That choice shapes most of the design.

I can turn these findings into a shareable review page, or edit them directly into `ARCHITECTURE.md`.