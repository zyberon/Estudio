---
tags: [capa, argo, summary]
---

# Certified Argo Project Associate (CAPA)

Summary of the KodeKloud prep course, with notes on the details that tend to show up in the exam.

1. The Argo Project at a glance|GitOps principles|Argo CD|Argo Workflows|Argo Rollouts|Argo Events|Exam cheat sheet

My own notes based on the course's public lesson notes (notes.kodekloud.com). Diagrams made for this summary.

# 1. The Argo Project at a glance

Argo is a family of Kubernetes-native tools, a **graduated CNCF project**. Each tool is driven by CRDs and controllers.

| Tool | Purpose | Key benefit |
|---|---|---|
| **Argo CD** | GitOps continuous delivery | Git is the single source of truth; detects and fixes drift |
| **Argo Workflows** | Container-native workflow engine (CI, batch, ML, data) | Steps and DAGs where every step runs as a pod |
| **Argo Rollouts** | Progressive delivery (canary, blue-green) | Metric-driven promotion and automatic rollback |
| **Argo Events** | Event-driven automation | Turns external events into Kubernetes actions |

![[capa-diagram-01.svg]]

*A typical end-to-end, event-driven pipeline built only from Argo tools.*

The course motivates it with the problems of push-based CI/CD: cluster credentials stored outside the cluster, configuration drift from manual changes, and unreliable disaster recovery because Git does not match reality.

# 2. GitOps principles

The OpenGitOps principles: the system is **declarative**, **versioned and immutable** (stored in Git), **pulled automatically** by an agent, and **continuously reconciled**.

| Concept | Meaning |
|---|---|
| Declarative | Describe the desired result, not the steps |
| Desired state | The manifests in Git |
| State store | Git: single source of truth, history, PR review, audit trail |
| Drift | Live state ≠ desired state (for example, someone ran `kubectl scale`) |
| Reconciliation | Observe → diff → act, in a continuous loop |
| Feedback loop | Monitoring (Prometheus, Grafana, Alertmanager) shows problems, and the fix is committed to Git |
| Rollback | `git revert` (keeps history), or the operator syncs back to a known-good commit |

**Push vs pull:** in push-based delivery, CI runs `kubectl apply` with credentials held outside the cluster. In pull-based delivery (GitOps), an agent inside the cluster pulls from Git, so credentials stay in the cluster.

# 3. Argo CD

## What it does

Argo CD is a declarative GitOps CD controller for Kubernetes. It watches applications, compares the **live state** with the **target state** in Git, shows the differences and syncs them manually or automatically. Supported sources: plain YAML/JSON, **Helm**, **Kustomize**, **Jsonnet** and Config Management Plugins (CMP). Ksonnet is deprecated.

Features: multi-cluster deployment; SSO (OIDC, OAuth2, SAML, LDAP, GitHub, GitLab, Microsoft) through Dex or a direct OIDC provider; RBAC and multi-tenancy; rollback to any commit; health assessment; drift visualization; Prometheus metrics; audit trail; sync hooks; Git webhooks; CLI; web UI.

## Architecture

| Component | Role |
|---|---|
| `argocd-server` | API server (gRPC and REST), web UI, authentication and RBAC enforcement |
| `argocd-repo-server` | Clones repos, caches them and **renders manifests** (Helm, Kustomize, Jsonnet, CMP) for a given revision |
| `argocd-application-controller` | Kubernetes controller that compares desired vs live state, detects drift and syncs (prune, self-heal) |
| `argocd-applicationset-controller` | Generates Applications from ApplicationSets |
| Redis | Cache only. It is disposable: persistent state lives in Kubernetes objects (Secrets, ConfigMaps, CRs) |
| Dex | Optional OIDC broker for SSO |
| `argocd-notifications-controller` | Sends notifications (Slack, Teams, email, webhooks) based on triggers and templates |

Git changes are detected by **polling** (every 3 minutes by default, set with `timeout.reconciliation` in `argocd-cm`; restart the repo-server after changing it) or by **webhooks** from GitHub, GitLab or Bitbucket for faster refreshes.

## Installation

| Option | Use |
|---|---|
| Core | Minimal, single user or team, no UI or SSO; labs |
| Multi-tenant non-HA: `install.yaml` (cluster-wide, cluster-admin) or `namespace-install.yaml` (namespace-scoped, mainly to manage remote clusters) | Proofs of concept and labs |
| Multi-tenant HA (`ha/install.yaml`) or the Helm chart `argo/argo-cd` | Production: multiple replicas, Redis HA |

The initial admin password is stored in the Secret `argocd-initial-admin-secret`. Log in with `argocd login <server>`.

## Core concepts

- **Application** (CR): links a **source** (`repoURL`, `path` or `chart`, `targetRevision`) to a **destination** (`server`, for example `https://kubernetes.default.svc` for the local cluster, plus `namespace`). It belongs to a `project` and has a `syncPolicy`. Create it with the CLI, the UI or a YAML manifest applied in the `argocd` namespace.
- **AppProject** (CR): groups Applications and sets limits: `sourceRepos`, `destinations`, `clusterResourceWhitelist` and blacklists, `roles` (with JWT tokens for automation) and `syncWindows`. The `default` project allows everything.
- **Refresh** re-reads Git and compares; it changes nothing. **Sync** applies the target state.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: color-app, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/org/app.git
    targetRevision: HEAD
    path: team-a/color-app
  destination:
    server: https://kubernetes.default.svc
    namespace: color
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

## Statuses

| Status | Values |
|---|---|
| Sync | Synced, OutOfSync |
| Health | Healthy, Progressing, Degraded, Suspended, Missing, Unknown |
| Operation | Running, Succeeded, Failed, Error, Terminating |

Built-in health checks: Deployments, StatefulSets and DaemonSets (replicas updated and available); Services of type LoadBalancer (an ingress IP or hostname is assigned); Ingress (status populated); PVCs (Bound); suspended CronJobs (Suspended). You can write **custom health checks in Lua** under `resource.customizations.health.<group_kind>` in `argocd-cm`; they return `hs.status` and `hs.message`.

## Sync strategies

| Policy | New file in Git | File removed from Git | Manual change in the cluster |
|---|---|---|---|
| Manual (no `automated` block) | Only when you sync | Only with a pruning sync | Shown as OutOfSync |
| `automated` | Applied | Kept in the cluster | Not reverted |
| + `prune: true` | Applied | **Deleted** from the cluster | Not reverted |
| + `selfHeal: true` | Applied | Kept (unless prune is on too) | **Reverted** to Git |

Automated sync is enabled by the **presence** of the `automated` block; there is no `enabled` flag. Useful `syncOptions`: `CreateNamespace=true`, `PruneLast=true`, `ApplyOutOfSyncOnly=true`, `ServerSideApply=true`, `Replace=true`. Per resource you can set `argocd.argoproj.io/sync-options: Prune=false`.

## Sync phases, hooks and waves

| Hook (`argocd.argoproj.io/hook`) | When it runs | Typical use |
|---|---|---|
| `PreSync` | Before the manifests are applied | Database migrations, backups |
| `Sync` | Together with the main resources | Complex rollout steps |
| `PostSync` | After everything is Healthy | Smoke tests, notifications |
| `SyncFail` | When the sync fails | Cleanup, alerting |
| `Skip` | Never applied |  |

- `argocd.argoproj.io/hook-delete-policy`: `HookSucceeded`, `HookFailed` or `BeforeHookCreation` (delete the old hook before creating a new one; avoids name clashes).
- **Sync waves** (`argocd.argoproj.io/sync-wave: "-1"`) order resources **within a phase**: lower numbers go first and equal numbers run in parallel. The default is **0**. Argo CD waits for each wave to be healthy before starting the next. Use negative waves for prerequisites and leave gaps between numbers (-10, -5, 0, 5) so you can insert steps later.

## Helm with Argo CD

The chart can come from a Git path or a Helm repository (`argocd repo add <url> --type helm`, then `--helm-chart <name> --revision <version>`). Values come from `valueFiles`, `values`/`valuesObject`, `parameters` or `--helm-set`. Argo CD renders the chart with `helm template` and applies the result, so **`helm list` does not show** releases managed by Argo CD.

## ApplicationSet

One template plus **generators** produces many Applications. Placeholders such as `{{.cluster}}` are filled in with each set of parameters (Go templates when `goTemplate: true`).

| Generator | Use |
|---|---|
| List | Fixed list of parameters |
| Cluster | One Application per cluster registered in Argo CD (filter by labels) |
| Git | One per directory or per file in a repo |
| Matrix | Cartesian product of two generators (apps × clusters) |
| Merge | Merge the parameters of several generators |
| SCM Provider | Repos found in a GitHub or GitLab organization |
| Pull Request | Temporary preview environments for each open PR |
| Cluster Decision Resource, Plugin | External logic |

# 4. Argo Workflows

Argo Workflows is a Kubernetes-native workflow engine. Every step runs as a **pod**, and a workflow is a `Workflow` CR. You use it for CI, batch jobs, data pipelines and ML. Features: steps and DAGs, conditions and loops, parameters and artifacts, retries, timeouts, suspend/resume, exit handlers, reusable templates, a UI and SDKs (Go, Python/Hera).

## Architecture and installation

- Control plane (the `argo` namespace): the **Workflow Controller** reconciles Workflow CRs and creates pods; the **Argo Server** provides the API and UI (port 2746).
- Execution plane: the workflow pods run in the user namespaces, under a `serviceAccountName` with the RBAC they need.
- Integrations: artifact repositories (S3, GCS, MinIO, HTTP…), a workflow archive in MySQL or PostgreSQL, OIDC SSO and Prometheus metrics.
- Install: the `quick-start-minimal` manifest for labs or `install.yaml` for production. It can be cluster-wide (the default) or namespace-scoped (`--namespaced`).
- CLI: `argo submit [--watch] wf.yaml -p message=hi --entrypoint other`, `argo list`, `argo get`, `argo logs`, `argo resubmit`, `argo retry`, `argo suspend`/`resume`, `argo stop`/`terminate`. `kubectl` also works on the CRs.

## The Workflow spec

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-        # unique suffix per submission
spec:
  entrypoint: main            # must match a template name
  arguments:
    parameters: [{ name: message, value: "hello" }]
  templates:
    - name: main
      inputs: { parameters: [{ name: message }] }
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]
```

| Template type | Does |
|---|---|
| `container` | Runs a container (the same fields as a pod container) |
| `script` | A container plus an inline `source`; the last line of stdout becomes `outputs.result` |
| `resource` | Creates, applies, patches or deletes a Kubernetes object; `successCondition` and `failureCondition` watch its status |
| `suspend` | Pauses for a `duration` or until someone resumes it (CLI, UI, API) |
| `containerSet` | Several containers in the same pod |
| `http` | Makes an HTTP request; the body goes to `outputs.result` |
| `plugin` | Executor plugins |
| `steps`, `dag` | **Invokers**: they don't do work themselves but compose other templates |

## Steps vs DAG

- **steps** is a list of lists: the outer groups run one after another, and the steps inside a group run in parallel.
- **dag**: tasks declare `dependencies: [A]` or `depends: "A && (B.Succeeded || C.Failed)"`. Independent tasks run in parallel, which gives fan-out and fan-in. `failFast` (default true) stops scheduling new tasks after a failure.

## Parameters, outputs and artifacts

- Workflow-level parameters go in `spec.arguments.parameters` and are overridden with `-p name=value` or `--parameter-file`. They are referenced as `{{workflow.parameters.x}}` and `{{inputs.parameters.x}}`.
- Outputs: `{{steps.generate.outputs.result}}` (script stdout), `outputs.parameters` (read from a file with `valueFrom.path`), or `{{tasks.X.outputs…}}` in a DAG.
- Artifacts are files. The producer declares `outputs.artifacts` with a `path`, the controller uploads it to the artifact repository, and the consumer declares `inputs.artifacts`, wired with `from: "{{steps.gen.outputs.artifacts.name}}"`. Configure an artifact repository first.
- **Artifact GC** (`artifactGC.strategy`): `OnWorkflowCompletion`, `OnWorkflowDeletion` or `Never`. Set it per workflow or per artifact.

## Reuse

| Resource | Scope | Use |
|---|---|---|
| `WorkflowTemplate` | Namespace | A library of templates for a team |
| `ClusterWorkflowTemplate` | Cluster | Shared by the platform team (`clusterScope: true` in `templateRef`) |
| `templateRef` | In a step | Call **one** template from the library |
| `workflowTemplateRef` | In the spec | Run the **whole** WorkflowTemplate |
| `CronWorkflow` | Namespace | Run on a cron schedule |

## Control fields

| Field | Purpose |
|---|---|
| `when` | Run a step only if a condition is true (`"{{steps.flip.outputs.result}} == heads"`) |
| `withItems`, `withParam`, `withSequence` | Loops: one pod per item, available as `{{item}}` |
| `parallelism` | Maximum number of concurrent pods (workflow or template level) |
| `retryStrategy` | `limit`, `retryPolicy` (Always, OnFailure, OnError, OnTransientError) and `backoff` |
| `activeDeadlineSeconds` | Timeout for the workflow or a template |
| `onExit` | Exit handler that always runs (cleanup, notifications); can read `{{workflow.status}}` |
| `daemon: true` | Background service (a database or mock API) for later steps; killed when the workflow ends |
| `ttlStrategy`, `podGC` | Clean up finished workflows and pods |
| `synchronization` | Mutexes and semaphores across workflows |

# 5. Argo Rollouts

Argo Rollouts is a controller plus the **`Rollout`** CRD, a drop-in replacement for a Deployment that adds **canary** and **blue-green** strategies, traffic shaping, metric analysis and automatic rollback. The controller manages a stable ReplicaSet and a new one. It does not proxy traffic itself: it configures an ingress controller or service mesh (NGINX, ALB, Istio, SMI, Traefik, Gateway API…) or, with no provider, approximates the split using replica counts.

## Kubernetes strategies vs progressive delivery

| Strategy | Downtime | Control | Use |
|---|---|---|---|
| RollingUpdate (Deployment) | None | `maxSurge` / `maxUnavailable` | Most stateless apps |
| Recreate (Deployment) | Yes | All at once | When two versions cannot run together |
| Blue-green (Rollout) | None (instant switch) | Whole environment swap | Validate before switching; instant rollback; costs double resources |
| Canary (Rollout) | None | Gradual traffic weights | Limit the blast radius with real feedback |

Related techniques: feature flags, A/B testing and traffic mirroring (shadowing).

## Installation

Cluster-wide (the default, needs cluster-admin) or namespace-scoped (`--namespaced`; a cluster admin installs the CRDs once). CLI: the `kubectl argo rollouts` plugin with `get rollout --watch`, `promote`, `abort`, `retry`, `undo`, `set image` and `dashboard`.

## Migrating a Deployment

- **Direct conversion**: change `kind: Deployment` to `apiVersion: argoproj.io/v1alpha1`, `kind: Rollout` and replace `strategy`. The pod template, `replicas` and `selector` stay the same.
- **`workloadRef`**: the Rollout references the existing Deployment, which keeps the pod template. `scaleDown` can be `never`, `onsuccess` or `progressively`. Existing CI/CD that patches the Deployment keeps working.

## Blue-green

| Field | Purpose |
|---|---|
| `activeService` (required) | Service that receives production traffic |
| `previewService` | Service for testing the new version before the switch |
| `previewReplicaCount` | Replicas of the new version while it is in preview |
| `autoPromotionEnabled` | Promote automatically (default **true**); set false to require `promote` |
| `autoPromotionSeconds` | Promote automatically after N seconds |
| `prePromotionAnalysis` / `postPromotionAnalysis` | Analysis before switching / after switching (failure rolls back) |
| `scaleDownDelaySeconds` | Keep the old version alive after the switch for a quick rollback (default 30s) |
| `abortScaleDownDelaySeconds` | Keep the new pods after an abort so you can debug them |

## Canary

```
strategy:
  canary:
    stableService: web-api-stable-svc
    canaryService: web-api-canary-svc
    trafficRouting: { nginx: { stableIngress: web-api-ingress } }
    analysis:                         # background analysis during the whole rollout
      templates: [{ templateName: success-rate }]
    steps:
      - setWeight: 10
      - pause: {}                     # indefinite: wait for promote
      - setWeight: 25
      - analysis:                     # inline analysis: blocks until it finishes
          templates: [{ templateName: check-api-error-rate }]
      - setWeight: 50
      - pause: { duration: 5m }
```

Other step types: `setCanaryScale` and `experiment`. If something fails the Rollout is **Degraded/aborted** and traffic goes back to stable. `maxSurge` and `maxUnavailable` still apply.

## AnalysisTemplate and AnalysisRun

- **AnalysisTemplate** (or `ClusterAnalysisTemplate`) is the reusable definition: `args` plus `metrics`. Each metric has a provider (**Prometheus**, Datadog, New Relic, CloudWatch, Wavefront, Kayenta, **Web** (HTTP), **Job**…) and `successCondition`/`failureCondition`, `interval`, `count`, `failureLimit`, `inconclusiveLimit` and `initialDelay`.
- **AnalysisRun** is one execution of a template. Its phases are Running, Successful, Failed, Error and Inconclusive. The Rollout controller watches the verdict and continues, pauses or rolls back. AnalysisRuns stay in the cluster as an audit trail.
- Where it runs: **background** (the whole canary), **inline** (a step that blocks), **pre- and post-promotion** (blue-green).
- An **Experiment** runs temporary ReplicaSets side by side (baseline vs canary) together with analysis.

```python
kind: AnalysisTemplate
spec:
  args: [{ name: service-name }]
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      failureLimit: 1
      successCondition: result[0] < 0.05
      provider:
        prometheus:
          address: http://prometheus:9090
          query: sum(rate(http_requests_total{service="{{args.service-name}}",code=~"5.."}[2m])) / sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

# 6. Argo Events

![[capa-diagram-02.svg]]

*Event flow in Argo Events.*

| CRD | Role |
|---|---|
| **EventSource** | Listens to external systems and turns each event into a **CloudEvent** (id, source, type, time, data). Types include `webhook` (port, endpoint, method), `calendar` (`interval` or cron `schedule`, plus `timezone`), `s3`/`minio` (bucket, events, prefix/suffix filter, credentials from a Secret), `github`, `gitlab`, `kafka`, `sqs`, `sns`, `pubsub`, `nats`, `resource` (Kubernetes objects)… |
| **EventBus** | The transport between EventSources and Sensors, usually named `default`. Backends: **NATS JetStream** (recommended), Kafka, or legacy NATS Streaming (STAN). Native (managed) or external (existing cluster). Production: 3 replicas, persistence and authentication |
| **Sensor** | Subscribes to the bus, declares `dependencies` (`name`, `eventSourceName`, `eventName`, optional `filters`) and runs `triggers` |

- Trigger types: Kubernetes object (create a Pod, Job or **Argo Workflow**), the Argo Workflow trigger, HTTP, AWS Lambda, Kafka, NATS, Slack, a custom trigger…
- `parameters` copy data from the event into the trigger: `src: {dependencyName: dep, dataKey: body.message}` → `dest: spec.containers.0.args.0`. Arrays use numeric indexes.
- With several dependencies, by default **all** are required (AND). Per trigger, `conditions: "(X || Y) && Z"` decides which dependencies fire it.
- Triggers support retries with backoff and rate limits. Keep EventSources light and do the heavy work in Workflows.

# Exam cheat sheet

| Topic | Remember |
|---|---|
| GitOps | Declarative, versioned and immutable, pulled automatically, continuously reconciled |
| Argo CD components | server (API/UI), repo-server (renders manifests), application-controller (reconciles), Redis (cache only), Dex (SSO), notifications, applicationset-controller |
| Detecting Git changes | Polling every 3 min (`timeout.reconciliation`) or webhooks |
| Statuses | Sync: Synced/OutOfSync · Health: Healthy/Progressing/Degraded/Suspended/Missing/Unknown |
| syncPolicy | The `automated` block turns auto-sync on · `prune` deletes what left Git · `selfHeal` reverts manual changes |
| Hooks and waves | PreSync → Sync → PostSync (SyncFail on failure) · waves: lower first, default 0, equal in parallel |
| AppProject | sourceRepos, destinations, cluster resource whitelist, roles, syncWindows |
| ApplicationSet | Generators: List, Cluster, Git, Matrix, Merge, SCM Provider, Pull Request, Plugin |
| Helm | Rendered with `helm template`, so `helm list` does not show the app |
| Workflows | Every step is a pod · steps = list of lists (outer sequential, inner parallel) · DAG with dependencies/depends · generateName |
| Workflow reuse | WorkflowTemplate (namespace), ClusterWorkflowTemplate, templateRef (one template) vs workflowTemplateRef (whole workflow), CronWorkflow |
| Script output | Last line of stdout → `outputs.result` |
| Rollouts | The Rollout CRD replaces a Deployment · blue-green: activeService, previewService, autoPromotionEnabled (default true) · canary: setWeight, pause, analysis |
| Analysis | AnalysisTemplate = definition, AnalysisRun = execution · background, inline, pre/post-promotion · Prometheus, Web, Job… |
| Events | EventSource → EventBus (JetStream) → Sensor → Trigger · CloudEvents · dependencies are ANDed by default |

