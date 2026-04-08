# Threat Analysis & Design Decisions

## 1. Threat Model

### Attacker Goals and Entry Points

The primary objective of an attacker targeting this environment is to exfiltrate cardholder data — specifically the plaintext card numbers managed by `token-vault-service`, or the PSP API credentials held by `psp-connector`. The realistic attack paths are:

**Compromised application pod.** A vulnerability in `payment-authorization-api` (e.g. RCE via a deserialization flaw or a dependency CVE) gives an attacker a shell inside the container. From there, they attempt lateral movement: reach `token-vault-service` directly, scrape secrets from the filesystem or environment, or pivot to the underlying node via a container escape.

**Supply chain / image tampering.** A malicious or backdoored container image is pushed to the internal registry and deployed. The image may attempt privilege escalation, exfiltrate secrets at startup, or establish an outbound C2 channel.

**Stolen PSP credential.** If a `psp-connector` pod is compromised, the attacker can extract Stripe or Adyen API keys from the pod environment and use them out-of-band to initiate fraudulent transactions against the payment processors directly.

**Insider / CI pipeline abuse.** A developer or pipeline with elevated cluster access deploys a workload that bypasses security controls, mounts host paths, or reads secrets it should not have access to.

### Blast Radius Per Service

| Service | Compromise Impact | PCI-DSS Scope |
|---|---|---|
| `payment-authorization-api` | Transaction visibility, pivot point to vault and PSP | In-scope |
| `token-vault-service` | Full cardholder data exposure — highest severity | In-scope, highest risk |
| `psp-connector` | PSP API key theft, fraudulent transaction initiation | In-scope |

`token-vault-service` is the crown jewel. Its compromise constitutes a full PCI-DSS breach event. Every control layer in this architecture is ultimately designed to make reaching it as hard as possible.

---

## 2. Controls Implemented

### Network Segmentation (PCI-DSS Req. 1)

A `default-deny-all` NetworkPolicy is applied to the entire `cde-production` namespace with both Ingress and Egress policy types. This means no pod can send or receive any traffic unless there is an explicit allow rule covering it. Each service has its own targeted policy:

`payment-authorization-api` accepts inbound traffic exclusively from the ingress gateway in the `ingress-nginx` namespace (enforced via `namespaceSelector` + `podSelector` conjunction). It can reach `token-vault-service` and `psp-connector` on port 8080/TCP only. No other egress is permitted.

`token-vault-service` accepts inbound on port 8080 only from `payment-authorization-api` and `reconciliation-job` (the batch reconciliation workload). It has no egress rules, meaning it cannot initiate any outbound connection — not to the internet, not to other pods.

`psp-connector` permits no inbound connections. Its sole egress is outbound HTTPS (443/TCP) to `0.0.0.0/0`, constrained to that single port. This is the only pod in the namespace that touches the internet, and it cannot be reached by anything inside or outside the cluster.

This architecture means a fully compromised `payment-authorization-api` pod cannot directly reach the internet, cannot reach any pod other than `token-vault-service` and `psp-connector`, and cannot receive connections from outside the namespace boundary.

### Pod Security Hardening (PCI-DSS Req. 2 & 6)

All three workloads are deployed under the Kubernetes `restricted` Pod Security Standard, enforced at the namespace level via the `pod-security.kubernetes.io/enforce: restricted` label on `cde-production`. The individual Deployment manifests reflect this explicitly:

Every container runs as a non-root UID (10001, 10002, 10003 respectively), with `runAsNonRoot: true` set at the pod spec level as a belt-and-suspenders control. Root filesystems are read-only across all three services; `payment-authorization-api` mounts an `emptyDir` volume at `/tmp` for any runtime writes the application needs. All Linux capabilities are dropped unconditionally. Privilege escalation is explicitly denied. Seccomp is set to `RuntimeDefault` on all pods, which blocks a large class of syscall-based container escape techniques.

Resource limits are defined on every container (CPU: 500m, memory: 512Mi), preventing a single compromised or malfunctioning pod from exhausting node resources and causing a denial-of-service condition across the CDE.

### Admission Control (PCI-DSS Req. 2 & 6)

Kyverno is deployed as a `ClusterPolicy` in `Enforce` mode with `background: true`. The following rules fire on every Pod admission request cluster-wide (excluding `kyverno` and `kube-system` system namespaces):

`disallow-root` rejects any pod that does not declare `runAsNonRoot: true` at the pod spec level. `disallow-root-container-override` uses a JMESPath deny condition to catch containers that explicitly set `runAsUser: 0`, closing the gap where a container-level override would otherwise circumvent the pod-level check.

`disallow-privileged` rejects any container with `privileged: true`. `disallow-hostpath` rejects any pod with a `hostPath` volume, which is one of the most reliable container escape vectors available to an attacker with pod scheduling capability.

`require-approved-registry` enforces that all images — including `initContainers` and `ephemeralContainers` — must originate from `registry.yuno-internal.io/*`. This closes the supply chain attack surface: no image from Docker Hub, GitHub Container Registry, or any other external source can be scheduled in the cluster without explicit policy modification.

`require-resources` rejects any pod that does not declare CPU and memory limits, preventing resource exhaustion attacks.

### Secrets Management (PCI-DSS Req. 3 & 8, Stretch Goal)

PSP API credentials are stored in HashiCorp Vault at `vault.yuno-internal.io` and synced into the `cde-production` namespace by the External Secrets Operator. The `ClusterSecretStore` authenticates to Vault using the Kubernetes auth method — the ESO service account's JWT is presented to Vault, which verifies it against the cluster's TokenReview API. There is no long-lived static token in the cluster.

The `ExternalSecret` resource syncs `stripe-api-key` and `adyen-api-key` every hour. The resulting Kubernetes Secret is owned by the ExternalSecret and its lifecycle is managed by ESO. The `psp-connector` deployment loads secrets via `envFrom.secretRef`, meaning credentials are only present in the pod's environment at runtime and are not embedded in any manifest checked into Git.

All secret access is auditable through Vault's audit log. Secret rotation in Vault propagates to pods on the next ESO sync cycle, after which a rolling restart picks up the new values.

---

## 3. Trade-offs: Security Strictness vs. Operational Velocity

### Controls Enforced Without Exception

**`require-approved-registry`** is a hard block with no escape hatch in the production namespace. The DevOps team's objection — "we needed a Redis fix from Docker Hub in 10 minutes" — is understood, but the correct response to that scenario is not a policy exception; it is a faster internal mirror process. The risk of pulling arbitrary images from Docker Hub into the CDE during an incident is unacceptable: a compromised or malicious image in the cardholder environment can exfiltrate live transaction data, and that outcome is categorically worse than a delayed hotfix.

**`disallow-hostpath` and `disallow-privileged`** are similarly non-negotiable. These are the two most reliable container escape vectors at an attacker's disposal. An exception here means a compromised pod can own the underlying node.

**`default-deny-all` NetworkPolicy** is never relaxed in `cde-production`. The blast radius of a compromised pod is directly bounded by this control.

### Where Exceptions Are Structured In

The emergency hotfix scenario is handled by architecture, not by policy relaxation. The correct model is:

A `cde-staging` or `cde-hotfix` namespace exists alongside `cde-production` with the same registry requirement but a faster image promotion pipeline. Engineers who need to test a third-party fix mirror the specific image to `registry.yuno-internal.io` (a process that should take under 5 minutes with a scripted push), validate it in staging, and then promote to production. This satisfies both the audit requirement and the operational velocity need.

For the read-only filesystem vs. application write requirement, `emptyDir` volumes are the sanctioned escape hatch. `payment-authorization-api` already uses one for `/tmp`. Any service that needs writable paths should declare specific `emptyDir` mounts for those paths rather than disabling `readOnlyRootFilesystem` entirely. This keeps the root filesystem immutable while giving the application what it needs.

### Compensating Controls for Any Relaxed Configuration

If a hotfix namespace is introduced with relaxed admission policies (e.g. temporarily allowing non-internal registry images under a break-glass procedure):

- All pod creation events in that namespace must generate alerts to the security team in real time.
- Pods in the hotfix namespace must be network-isolated from `cde-production` via NetworkPolicy — no pod in `cde-hotfix` can reach `token-vault-service` under any circumstances.
- Time-boxing: the relaxed policy must automatically revert or expire after a defined window (e.g. 4 hours), enforced via a Kyverno `CleanupPolicy` or an operator-managed TTL.
- Every use of the break-glass procedure is logged, reviewed post-incident, and treated as a finding in the next PCI-DSS audit cycle.

---

## 4. Residual Risks

**No mTLS between services.** All in-cluster service-to-service traffic is currently plaintext. NetworkPolicies enforce which pods can communicate, but they do not authenticate the identity of the caller at the protocol level. A pod that gains a valid IP and port (e.g. via ARP spoofing on a misconfigured CNI, or through a future CVE) could impersonate `payment-authorization-api` and call `token-vault-service` without being detected. Mutual TLS via a service mesh (Istio or Cilium) would close this gap.

**No runtime behavioural detection.** The current controls are all preventive. There is no deployed runtime security agent (Falco or equivalent) monitoring for anomalous syscalls, unexpected process executions, or outbound connection attempts. An attacker who finds a zero-day inside a container after admission would not trigger any alert until the NetworkPolicy blocked their lateral movement — and even that block is silent.

**`token-vault-service` has no egress policy.** The current `token-vault-policy.yaml` defines only ingress rules. While the `default-deny-all` policy ensures no egress is permitted, a defence-in-depth posture calls for an explicit egress allowlist on the vault service itself (e.g. only allowing outbound connections to the encrypted database backing store). This explicit egress rule is missing and should be added once the backing database endpoint is known.

**Secrets still land as Kubernetes Secrets.** Even with ESO + Vault, the synced PSP credentials exist as native Kubernetes Secret objects (base64-encoded) in the `cde-production` namespace. Without KMS envelope encryption at rest (etcd encryption), anyone with direct etcd access can read them. RBAC restricts API-level access, but etcd-level access bypasses RBAC entirely.

**Image tags are mutable.** All three workloads use `:latest` tags. If a compromised image is pushed to `registry.yuno-internal.io` with the same tag, Kubernetes will pull it on the next pod restart without any change to the manifest. Pinning to immutable digest references (e.g. `image@sha256:...`) and enforcing this via a Kyverno rule would eliminate this vector.

**No RBAC ServiceAccounts defined.** The workloads do not specify `serviceAccountName`, meaning they use the `default` service account. In a default cluster configuration the default service account has no permissions, but this should be made explicit: each service should have a dedicated ServiceAccount with a minimal RBAC Role bound to it, and `automountServiceAccountToken: false` should be set where the application does not call the Kubernetes API.

---

## 5. Future Work (Next 2 Weeks)

**Service mesh with mTLS.** Deploy Cilium or Istio to enforce mutual TLS between all services in `cde-production`. This upgrades service identity from "correct IP and port" to "verified certificate signed by the cluster CA", closing the protocol-level impersonation gap.

**Falco runtime security.** Deploy Falco with CDE-specific rules: alert on any process execution inside a container that is not the entrypoint binary, alert on any outbound connection attempt that does not match expected destinations, alert on any file access outside declared volume mounts. These rules catch post-compromise behaviour before data is exfiltrated.

**Kubernetes API audit policy.** Configure the API server to emit audit events for all reads and writes to Secrets in `cde-production`, all pod creation events, and all RBAC binding changes. Forward audit logs to a tamper-evident store (e.g. CloudTrail, a SIEM, or an immutable S3 bucket). This satisfies PCI-DSS Requirement 10 (audit trails).

**Dedicated ServiceAccounts with minimal RBAC.** Create a dedicated ServiceAccount per workload. Bind only the Roles actually needed (likely none for most services). Set `automountServiceAccountToken: false` on all three Deployments to prevent the mounted JWT from being usable for API server calls.

**Immutable image digests.** Replace `:latest` tags with pinned `@sha256` digests in all Deployment manifests. Add a Kyverno rule that rejects any image reference using a mutable tag.

**KMS encryption for etcd.** Enable envelope encryption for Kubernetes Secrets at rest using a cloud KMS key (AWS KMS, GCP KMS, or equivalent). This ensures that even raw etcd access does not yield readable secret values.

**Automated secret rotation testing.** Implement a scheduled job that rotates a canary PSP credential in Vault and verifies that the ESO sync and pod rolling restart complete within an expected time window. This validates the rotation pipeline before a real rotation is needed under pressure.
