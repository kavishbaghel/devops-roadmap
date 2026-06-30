# Part 8: Helm and Configuration Management

> **Prerequisite**: Part 3.4.4 (ConfigMap immutability and rollout-triggering patterns).
> **Goal**: Be able to author production-grade Helm charts and make an informed Helm-vs-Kustomize decision per use case, rather than defaulting to whichever tool you learned first.

---

## 8.1 Helm Fundamentals

### 8.1.1 Charts, Templates, Values, Releases

Helm is, at its core, a **templating engine plus a package manager** for Kubernetes manifests. Four concepts compose the model:

- **Chart**: a packaged collection of templated Kubernetes manifests, plus metadata — the unit of distribution (analogous to an `apt`/`npm` package).
- **Template**: a YAML manifest with Go-template directives (`{{ }}`) for value substitution, conditionals, and loops.
- **Values**: the configuration input (`values.yaml`, or `--set`/`-f` overrides at install time) that fills in template placeholders.
- **Release**: a specific, named, *deployed instance* of a chart with a specific set of values into a specific cluster/namespace — the same chart can produce many independent releases (e.g., `backend-staging` and `backend-prod`, both from the same chart, with different values).

```
mychart/
├── Chart.yaml              # name, version, dependencies metadata
├── values.yaml              # default configuration values
├── values.schema.json       # optional - JSON Schema validation for values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl         # reusable named template snippets
│   └── NOTES.txt            # post-install message shown to the user
└── charts/                  # vendored subchart dependencies
```

```yaml
# Chart.yaml
apiVersion: v2
name: sre-bot-agent
description: SRE Bot AI agent deployment
version: 1.4.0          # the CHART version (semver, your own release cadence)
appVersion: "2.1.0"      # the underlying APPLICATION version being deployed — independent of chart version
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: ghcr.io/kavishbaghel/sre-bot-agent
  tag: "2.1.0"
  pullPolicy: IfNotPresent
resources:
  requests: {cpu: 250m, memory: 256Mi}
  limits: {cpu: 500m, memory: 512Mi}
ollama:
  endpoint: "http://ollama.sre-bot-infra.svc.cluster.local:11434"
  model: "llama3.2:3b"
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "sre-bot-agent.fullname" . }}
  labels:
    {{- include "sre-bot-agent.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "sre-bot-agent.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "sre-bot-agent.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          env:
            - name: OLLAMA_ENDPOINT
              value: {{ .Values.ollama.endpoint | quote }}
            - name: OLLAMA_MODEL
              value: {{ .Values.ollama.model | quote }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
helm install sre-bot-agent ./mychart -f values-prod.yaml --namespace sre-bot-apps
helm upgrade sre-bot-agent ./mychart -f values-prod.yaml --namespace sre-bot-apps
helm rollback sre-bot-agent 1 --namespace sre-bot-apps    # roll back to revision 1
helm history sre-bot-agent --namespace sre-bot-apps
helm template ./mychart -f values-prod.yaml               # render locally WITHOUT installing — essential for review
helm diff upgrade sre-bot-agent ./mychart -f values-prod.yaml   # requires the helm-diff plugin — shows exact change before applying
```

**`helm template` and `helm diff` are the two commands most worth building habit around** — rendering manifests locally before any install/upgrade, and diffing against the live cluster state before applying, turns Helm changes from "deploy and hope" into a reviewable, predictable operation, which matters enormously for peer review and for catching templating mistakes (a misplaced `nindent`, a missing default) before they hit a cluster.

### 8.1.2 Chart Dependencies and Library Charts

```yaml
# Chart.yaml - declaring a dependency on a subchart (e.g., a shared Postgres chart from Bitnami)
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled    # only include this subchart if values.postgresql.enabled is true
```

```bash
helm dependency update ./mychart    # fetches subcharts into charts/ based on Chart.yaml
```

Values for a subchart are namespaced under its own key in the parent's `values.yaml`:

```yaml
# Parent chart's values.yaml
postgresql:
  enabled: true
  auth:
    database: srebot
  primary:
    persistence:
      size: 50Gi
```

**Library charts** are a distinct chart type (`type: library` in `Chart.yaml`) that produce **no deployable resources of their own** — they exist purely to define reusable named templates (`_helpers.tpl`-style snippets) that *other* charts import, useful for standardizing common patterns (e.g., a consistent label/annotation scheme, or a standard PodDisruptionBudget template) across many charts maintained by the same platform team, without copy-pasting the same boilerplate into every chart.

### 8.1.3 Helm Hooks and Lifecycle Management

Helm hooks let specific resources execute at defined points in a release's lifecycle, **outside** the normal install/upgrade resource-apply ordering — most commonly used for database migrations that must run *before* the new application version's Pods start.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "sre-bot-agent.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded,before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate.sh"]
```

| Hook | Fires |
|---|---|
| `pre-install` / `pre-upgrade` | Before any chart resources are created/updated — the standard slot for migrations |
| `post-install` / `post-upgrade` | After all resources are created/updated and (for Deployments) considered ready |
| `pre-delete` / `post-delete` | Around `helm uninstall` — useful for graceful cleanup of external resources |
| `pre-rollback` / `post-rollback` | Around `helm rollback` |

`hook-weight` controls ordering among multiple hooks at the same lifecycle point (lower runs first); `hook-delete-policy` controls cleanup of the hook resource itself (`hook-succeeded` deletes it only on success, leaving failed migration Jobs around for debugging — generally the safer default over silently deleting evidence of a failure).

**Important limitation worth knowing**: hook resources are **not tracked as part of the release's normal manifest set** in the same way regular templated resources are — they don't show up in `helm get manifest` the same way, and aren't automatically cleaned up on `helm uninstall` unless a `pre-delete`/`post-delete` hook is explicitly defined for that purpose. This trips up engineers who expect `helm uninstall` to remove *everything* a chart ever created.

---

## 8.2 Beyond Helm

### 8.2.1 Kustomize: Overlays and Patches

Kustomize takes a fundamentally different philosophy from Helm: **no templating language at all** — instead, you maintain a `base` of plain, valid Kubernetes YAML, and define `overlays` that **patch** the base for each environment using strategic merge patches or JSON patches.

```
myapp/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
    target:
      kind: Deployment
      name: backend
configMapGenerator:                 # see Part 3.4.4 — automatic content-hash suffixing
  - name: app-config
    literals:
      - LOG_LEVEL=warn
images:
  - name: backend
    newTag: "2.1.0"
```

```yaml
# overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 6
```

```bash
kubectl kustomize overlays/production/      # render only, no apply
kubectl apply -k overlays/production/        # render AND apply, in one step (kubectl has native kustomize support)
```

Because Kustomize's base is always **plain, valid YAML** (never templated, never has placeholder syntax that would fail `kubectl apply -f` directly), it's trivially easy to review, diff with standard YAML/git tooling, and reason about without learning any templating language — this is its core appeal over Helm for teams that find Go-template syntax (`{{- if .Values.x }}`, whitespace control via `nindent`) adds more cognitive overhead than value, especially for simpler multi-environment promotion needs.

### 8.2.2 GitOps-Native Config Strategies (Helm vs Kustomize vs Raw Manifests)

| Dimension | Helm | Kustomize | Raw manifests |
|---|---|---|---|
| Templating power | Full Go-template (conditionals, loops, functions) | None — patch-based only | None |
| Distribution/packaging | First-class (chart repos, versioned packages, public chart ecosystem like Bitnami) | No native packaging concept | No native packaging concept |
| Learning curve | Steeper (Go templates, chart structure conventions) | Gentler (it's still just YAML) | None — but doesn't scale to multi-env |
| Multi-environment | Via separate `values-<env>.yaml` files | Native, first-class concept (`overlays/`) | Manual duplication, error-prone |
| Built into `kubectl` | No (separate binary required) | Yes (`kubectl apply -k`) | N/A |
| Release/rollback tracking | First-class (`helm rollback`, release history) | None natively (relies on Git history + your GitOps tool's own tracking) | None |

**The practical decision framework**, relevant directly to your sre-bot Helm chart work and any future chart-authoring decisions:

- **Choose Helm** when you're packaging something for **reuse by others** (a chart consumed by multiple teams or published externally), when you need genuinely complex conditional logic (e.g., "render this block only if both feature flags are set and we're in this cloud provider"), or when you want built-in release/rollback tracking independent of your GitOps tool.
- **Choose Kustomize** when you have a **small, fixed set of known environments** and the differences between them are simple, additive patches (replica counts, resource limits, a handful of env-specific config values) — and especially when you want the lowest possible barrier for a new team member to read and understand exactly what gets deployed without learning a templating DSL first.
- **Many real-world GitOps setups use both together**: Helm for packaging genuinely reusable/complex charts (including third-party charts like the Bitnami Postgres dependency from 8.1.2), with Kustomize patches layered on top for final environment-specific tweaks — ArgoCD natively supports this combined pattern (`kustomize.components` referencing a Helm-rendered base), which is worth knowing exists rather than assuming the two tools are mutually exclusive.

This decision directly sets up Part 14 (GitOps and CI/CD) — both ArgoCD and Flux treat Helm charts and Kustomize overlays as first-class, interchangeable application sources, so the choice here is genuinely about authoring ergonomics and reuse needs, not about GitOps tool compatibility.

---

## Summary: What You Should Take Away From Part 8

- Helm separates Chart (template), Values (config input), and Release (a specific deployed instance) — the same chart, different values, produces independent releases.
- `helm template` and `helm diff` should be habitual pre-flight checks before any install/upgrade — they turn Helm from "deploy and hope" into a reviewable operation.
- Library charts exist specifically to share reusable template snippets across multiple charts without copy-paste duplication — valuable for a platform team standardizing patterns across many services.
- Helm hooks run outside normal resource-apply ordering and are not cleaned up automatically on uninstall unless explicitly handled — a common source of "why didn't uninstall remove everything" confusion.
- Kustomize's no-templating, patch-based philosophy trades expressive power for review-simplicity — bases stay plain, valid YAML at every layer.
- The Helm-vs-Kustomize choice is not either/or in practice; many GitOps setups layer Kustomize patches on top of Helm-rendered bases, and both tools integrate natively with ArgoCD/Flux.

**Next**: Parts 1–8 form the complete core-Kubernetes-fundamentals arc. Part 9 begins the Observability series (Foundations: the three pillars, SLIs/SLOs/error budgets, instrumentation philosophy) — directly applicable to your current sre-bot pipeline work.
