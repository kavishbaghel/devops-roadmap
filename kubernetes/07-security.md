# Part 7: Security

> **Prerequisite**: Part 2.3.3 (API request flow: authn/authz/admission), Part 3.4.2 (Secrets limitations).
> **Goal**: Understand identity, policy enforcement, and hardening deeply enough to design a defensible multi-tenant cluster.

---

## 7.1 Authentication and Authorization

### 7.1.1 ServiceAccounts and Token Projection

Every Pod runs **as** a ServiceAccount (defaulting to the namespace's `default` ServiceAccount if unspecified) — this is the identity the Pod presents when it calls the Kubernetes API itself (e.g., a controller watching resources, or any application that needs to introspect/manage cluster state).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sre-bot-agent
  namespace: sre-bot-apps
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: sre-bot-agent
      automountServiceAccountToken: true   # explicit is better than relying on default
```

**Token projection** (the modern mechanism, replacing the older long-lived, auto-mounted Secret-based ServiceAccount tokens): the kubelet mounts a **time-bound, audience-bound JWT** into the Pod's filesystem (`/var/run/secrets/kubernetes.io/serviceaccount/token`), automatically refreshed before expiry, rather than a static token that never expires and is valid for any audience.

```yaml
volumes:
  - name: api-token
    projected:
      sources:
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600    # short-lived, auto-refreshed
            audience: api               # token only valid for this specific audience, not reusable elsewhere
```

This matters significantly for security posture: a leaked **legacy** ServiceAccount token Secret was valid forever, with no audience restriction, until manually rotated — a leaked **projected** token is valid for at most an hour and only for its declared audience, dramatically shrinking the blast radius of credential leakage. Modern Kubernetes defaults to projected tokens automatically; explicitly disabling `automountServiceAccountToken` for Pods that don't need API access at all is still good hygiene (smallest possible attack surface).

### 7.1.2 RBAC: Roles, ClusterRoles, Bindings

Four objects implement Kubernetes' native authorization model:

```yaml
# Role - namespaced permission set
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: sre-bot-apps
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
# RoleBinding - grants a Role to a subject, WITHIN that namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sre-bot-agent-pod-reader
  namespace: sre-bot-apps
subjects:
  - kind: ServiceAccount
    name: sre-bot-agent
    namespace: sre-bot-apps
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ClusterRole - cluster-scoped permission set (can also be used for namespaced resources,
# bound via a RoleBinding, for permissions that need to be reused across MANY namespaces)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]      # nodes are cluster-scoped — ONLY a ClusterRole can grant access to them
    verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding - grants cluster-wide, across ALL namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-node-viewer
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

**The four-combination matrix worth memorizing**:

| Role type | Binding type | Effective scope |
|---|---|---|
| `Role` | `RoleBinding` (same namespace) | Permissions apply only within that one namespace |
| `ClusterRole` | `RoleBinding` | Cluster-scoped *permission set*, but granted only within the binding's namespace — the standard pattern for reusing a common permission set (e.g., "view" or "edit") across many namespaces without duplicating Role definitions |
| `ClusterRole` | `ClusterRoleBinding` | Permissions apply cluster-wide, across all namespaces |
| `Role` | `ClusterRoleBinding` | **Invalid** — not a supported combination |

**RBAC is purely additive (allow-only)** — there is no `deny` rule type in native Kubernetes RBAC. This is a frequent point of confusion for engineers coming from systems with explicit deny rules; the only way to "restrict" access in native RBAC is to simply not grant a permission, or to use a separate policy engine (Part 7.2) for more expressive deny/conditional logic.

```bash
# Debugging RBAC — "can I do X" is the essential first diagnostic command
kubectl auth can-i create pods --namespace=sre-bot-apps --as=system:serviceaccount:sre-bot-apps:sre-bot-agent
kubectl auth can-i --list --as=system:serviceaccount:sre-bot-apps:sre-bot-agent -n sre-bot-apps
```

### 7.1.3 Impersonation and Aggregated ClusterRoles

**Impersonation**: lets a user/ServiceAccount with the `impersonate` verb act *as* another identity for a request — used heavily by CI/CD systems and platform tooling that need to perform an action "as if" a specific user did it (preserving accurate audit trails) rather than acting under the tool's own broad service identity.

```yaml
rules:
  - apiGroups: [""]
    resources: ["users", "groups", "serviceaccounts"]
    verbs: ["impersonate"]
```

```bash
kubectl get pods --as=jane@example.com --as-group=platform-team
```

**Aggregated ClusterRoles**: a mechanism for *composing* ClusterRoles dynamically via label selectors, rather than maintaining one giant monolithic rule list — particularly useful when extending built-in roles (`view`, `edit`, `admin`) to include permissions for Custom Resources without editing the built-in role definitions directly.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre-bot-crd-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"    # this label is the magic —
rules:                                                       # built-in "view" ClusterRole auto-aggregates this
  - apiGroups: ["srebot.io"]
    resources: ["incidents", "diagnostics"]
    verbs: ["get", "list", "watch"]
```

This pattern is directly relevant if you build CRDs for sre-bot (Part 16.1) — by labeling a ClusterRole with `rbac.authorization.k8s.io/aggregate-to-view: "true"`, any user/ServiceAccount already bound to the built-in `view` ClusterRole automatically gains read access to your custom `incidents`/`diagnostics` resources too, without anyone needing to update the built-in role or maintain a separate binding.

---

## 7.2 Admission Control

### 7.2.1 Validating and Mutating Webhooks

Already introduced structurally in Part 2.3.3. Both webhook types are configured via dedicated objects pointing at an HTTPS endpoint (typically a Service backed by your own controller/Pod) that the API server calls synchronously during the admission phase of every matching request.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-resource-limits
webhooks:
  - name: require-limits.policy.example.com
    clientConfig:
      service:
        name: policy-webhook
        namespace: policy-system
        path: "/validate"
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    failurePolicy: Fail
    sideEffects: None
    admissionReviewVersions: ["v1"]
    timeoutSeconds: 5
```

The webhook server receives an `AdmissionReview` request (containing the full object being created/updated) and must respond within `timeoutSeconds` with an `allowed: true/false` decision (and, for mutating webhooks, an optional JSON patch to apply).

### 7.2.2 Policy Engines: OPA/Gatekeeper, Kyverno

Writing and operating raw admission webhooks by hand for every policy is heavy — two dominant policy-as-code engines abstract this:

**OPA/Gatekeeper**: built on the general-purpose **Open Policy Agent**, using the **Rego** policy language. Policies are defined as `ConstraintTemplate` objects (the reusable Rego logic) and `Constraint` instances (applying that template with specific parameters):

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits
          msg := sprintf("Container %v has no resource limits", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
```

**Kyverno**: Kubernetes-native, policies written as plain YAML (no separate language to learn) — generally faster to adopt for teams without prior Rego experience.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce    # Enforce (block) | Audit (log only, don't block)
  rules:
    - name: check-resource-limits
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Resource limits are required on all containers."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

Kyverno also supports **mutation** and **generation** policies natively (e.g., automatically injecting a default NetworkPolicy into every new namespace) — a capability that, with Gatekeeper, requires combining its validating-only model with a separate mutating-webhook mechanism. The Rego-vs-YAML tradeoff is the most commonly cited deciding factor: Gatekeeper's Rego is more expressive for genuinely complex cross-resource logic; Kyverno's YAML is faster to read, write, and review for the majority of common policies, which matters directly for peer-reviewability in a platform team context.

### 7.2.3 Pod Security Standards/Admission

Kubernetes 1.25 removed the older **PodSecurityPolicy** (deprecated for being notoriously confusing to operate) in favor of the simpler, built-in **Pod Security Admission (PSA)** controller, enforcing one of three predefined **Pod Security Standards** at the namespace level via labels — no separate policy engine required for baseline hardening:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

| Standard | Posture |
|---|---|
| `privileged` | Unrestricted — equivalent to no policy at all |
| `baseline` | Blocks known privilege-escalation vectors (e.g., host namespaces, privileged containers) while remaining broadly compatible with common workloads |
| `restricted` | Heavily hardened — requires running as non-root, drops all Linux capabilities by default, enforces seccomp, disallows privilege escalation entirely |

PSA is intentionally **simpler but less expressive** than Gatekeeper/Kyverno — it covers a fixed, well-known set of Pod-level security concerns out of the box with zero custom policy authoring, while Gatekeeper/Kyverno are needed for anything beyond that fixed set (custom organizational rules, non-Pod resource policies, mutation/generation). Most mature platform teams run **both**: PSA at `restricted` as a baseline floor (simple, built-in, hard to misconfigure), layered with Kyverno/Gatekeeper for organization-specific policy beyond what PSA covers.

---

## 7.3 Workload and Cluster Hardening

### 7.3.1 `securityContext`, seccomp, AppArmor

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:                      # Pod-level — applies to all containers
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:                  # Container-level — overrides/refines pod-level
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]     # only add back the specific capability genuinely needed
```

- **`runAsNonRoot` / `runAsUser`**: prevents running as UID 0 inside the container — critical because a container running as root, combined with any container-escape vulnerability, gives an attacker root on the *host* far more easily than a non-root container would.
- **`readOnlyRootFilesystem: true`**: makes the container's root filesystem immutable at runtime — a powerful, simple hardening step (an attacker who achieves code execution can't write a persistent payload to disk) that's compatible with the vast majority of well-designed stateless applications (which should be writing only to mounted volumes or `emptyDir`, never to their own image layer, anyway).
- **`capabilities.drop: ["ALL"]`**: Linux capabilities are fine-grained privilege units (`NET_BIND_SERVICE`, `SYS_ADMIN`, `NET_RAW`, etc.) that traditionally come bundled wholesale with "root" — dropping all and adding back only the specific ones genuinely required follows the principle of least privilege precisely, rather than the all-or-nothing root/non-root binary alone.
- **`seccompProfile: RuntimeDefault`**: enables the container runtime's default seccomp filter, blocking a substantial set of rarely-needed, historically exploit-prone syscalls — a meaningful hardening step with essentially zero compatibility cost for typical application workloads.
- **AppArmor**: a complementary Linux Security Module providing path-based access control (which files/capabilities a process may use), configured via Pod annotations referencing a profile pre-loaded on the node — less universally used than seccomp in cloud-native contexts but still relevant on AppArmor-based distributions (notably Ubuntu-based nodes).

### 7.3.2 Image Scanning and Supply Chain Security (SBOM, Signing, Cosign)

Modern supply-chain security for container images rests on three complementary practices:

**Vulnerability scanning** (Trivy, Grype, or cloud-native equivalents like ECR/GAR scanning) inspects image layers against known CVE databases — typically integrated as a CI pipeline gate (Part 14.2) blocking deployment of images with critical/high unpatched vulnerabilities.

```bash
trivy image myapp:1.0 --severity HIGH,CRITICAL --exit-code 1
```

**Software Bill of Materials (SBOM)**: a structured, machine-readable manifest of every package/dependency/library that went into building an image — increasingly a regulatory/compliance requirement (e.g., for vendors selling into regulated industries or government), and operationally valuable for rapidly answering "are we affected by this newly-disclosed CVE" across an entire fleet of images without re-scanning everything from scratch.

```bash
syft myapp:1.0 -o spdx-json > myapp-sbom.json
```

**Image signing** (the supply-chain-attack mitigation — preventing an attacker who compromises a registry, or performs a man-in-the-middle, from substituting a malicious image that still "looks" legitimate): **cosign** (part of the Sigstore project) is the dominant tool, supporting both traditional key-pair signing and **keyless signing** (using short-lived certificates issued via OIDC identity, eliminating long-term private key management entirely).

```bash
# Sign an image (keyless, using OIDC identity — e.g., your CI system's identity)
cosign sign myregistry.io/myapp:1.0

# Verify before deployment
cosign verify myregistry.io/myapp:1.0 \
  --certificate-identity=https://github.com/kavishbaghel/sre-bot/.github/workflows/build.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com
```

**Enforcement at admission time**: signature verification is only meaningful if it's actually *enforced*, not merely available — this is typically wired up as a Kyverno or Gatekeeper policy rejecting any image whose signature can't be verified against a trusted identity, closing the loop between "we sign our images" and "the cluster actually refuses to run unsigned ones":

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences: ["myregistry.io/*"]
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/kavishbaghel/sre-bot/*"
```

### 7.3.3 Secrets Management Integration (Vault, External Secrets Operator)

Covered in depth in Part 3.4.2 — included here as a cross-reference, since secrets management is as much a cluster-hardening concern as it is a configuration-management one. The key hardening principle worth restating: native Kubernetes Secrets should be treated as a **delivery mechanism** synced from an external system of record (Vault, cloud secret managers), not as the system of record itself — centralizing rotation, audit logging, and fine-grained access policy outside of etcd's relatively coarse RBAC-only access model.

### 7.3.4 mTLS via Service Mesh

A brief forward-reference to Part 13: a service mesh's automatic mutual TLS provides **encryption in transit** and **strong workload identity** (cryptographic proof of "which service is actually calling," independent of and stronger than IP-based trust) for all pod-to-pod traffic, without requiring application code changes. This is the most common production answer to "how do we ensure pod-to-pod traffic is encrypted by default" — covered fully, with traffic management and zero-trust architecture, in Part 13.2.

### 7.3.5 Runtime Security and Threat Detection (Falco)

Admission control (7.2) and hardening (7.3.1) reduce attack surface and prevent known-bad configurations from being deployed — but neither detects **active exploitation happening right now** inside an already-running container. **Falco** (a CNCF graduated project) addresses this gap: a runtime security agent (typically a DaemonSet) consuming kernel-level syscall events (via eBPF, in modern deployments) and evaluating them against a rule set for suspicious behavior.

```yaml
# Example Falco rule - simplified, illustrating the detection model
- rule: Unexpected shell spawned in container
  desc: Detect a shell being spawned inside a container at runtime (often a sign of exploitation)
  condition: >
    spawned_process and container
    and shell_procs
    and not proc.pname in (known_shell_spawners)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

Common Falco detections relevant to Kubernetes specifically: unexpected shell execution inside a container that should never need one (especially relevant for `readOnlyRootFilesystem` + distroless images, where any shell spawn at all is inherently suspicious), writes to sensitive host paths via a misconfigured `hostPath` mount, outbound connections to known-malicious IPs, and privilege escalation attempts (e.g., a process unexpectedly attempting to load a kernel module). Falco alerts integrate naturally into the broader observability pipeline (Part 10–12) — typically shipped as structured logs/events into the same pipeline as application logs, or alerted on directly via its own sidecar/exporter.

### 7.3.6 Network Egress Control and Firewalling at the Node Level

NetworkPolicy (Part 4.4) governs Pod-to-Pod and Pod-to-external traffic from *within* the cluster's CNI layer — but a genuinely defense-in-depth posture also constrains egress at the **node/infrastructure level**, independent of and beneath whatever the CNI enforces, as a backstop against CNI misconfiguration or a compromised node attempting to exfiltrate data directly.

In cloud environments, this typically means: **security groups/firewall rules** on the node's underlying VPC/subnet restricting which external IP ranges nodes can reach at all (e.g., only specific package registries, cloud APIs, and explicitly approved third-party endpoints — denying broad internet egress by default), combined with **egress gateways** (a dedicated, tightly-controlled set of nodes/NAT gateways that *all* cluster egress is forced through, providing a single chokepoint for both firewalling and audit logging of outbound traffic) rather than allowing every node to originate arbitrary outbound connections directly. Cilium's `CiliumClusterwideNetworkPolicy` and dedicated egress-gateway features (covered further in Part 13's mesh material) increasingly let teams implement this pattern natively within the CNI layer itself, rather than relying solely on cloud-provider firewall constructs.

---

## Summary: What You Should Take Away From Part 7

- Projected, time-bound, audience-scoped ServiceAccount tokens are a meaningful security upgrade over legacy long-lived tokens — leaked-credential blast radius shrinks from "forever, anywhere" to "an hour, one specific audience."
- RBAC is purely additive — there is no native deny rule, and the four Role/Binding combinations (only one of which is invalid) should be memorized precisely, not approximated.
- Admission control (mutating then validating) runs strictly after authn/authz succeed — a request can be fully authorized and still rejected at this final stage, a frequent source of debugging confusion.
- Pod Security Admission gives simple, built-in baseline hardening with zero custom policy authoring; Gatekeeper/Kyverno are needed for anything organization-specific beyond PSA's fixed standards — mature setups run both, layered.
- `readOnlyRootFilesystem` and `capabilities.drop: ["ALL"]` are high-value, low-compatibility-cost hardening defaults that should be the starting point, not an afterthought, for new workloads.
- Image signing (cosign) is only meaningful as a security control when actually *enforced* at admission time, not merely practiced as a CI step nobody verifies downstream.
- Runtime security (Falco) fills the detection gap that admission control and static hardening cannot: catching active exploitation in an already-running, already-"compliant" container.
- Defense-in-depth means NetworkPolicy at the CNI layer *and* node/infrastructure-level egress firewalling — neither alone is a complete answer.

**Next**: Part 8 covers Helm and configuration management — chart authoring, lifecycle hooks, and the GitOps-adjacent comparison between Helm and Kustomize that sets up Part 14's CI/CD material.
