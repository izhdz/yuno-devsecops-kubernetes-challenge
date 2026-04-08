# Yuno DevSecOps — Kubernetes CDE Hardening Challenge

This repository contains the Kubernetes hardening configuration for Yuno's Cardholder Data Environment (CDE), implemented as part of a DevSecOps technical challenge. The goal is a PCI-DSS-aligned, defence-in-depth configuration for a production-grade payment processing cluster.

## Repository Structure

```
.
├── base/
│   ├── namespace.yaml              # cde-production namespace with PSS: restricted label
│   └── resource-quotas.yaml        # Cluster resource caps for the CDE namespace
│
├── network-policies/
│   ├── default-deny.yaml           # Zero-trust baseline: deny all ingress + egress
│   ├── payment-api-policy.yaml     # payment-authorization-api allow rules
│   ├── token-vault-policy.yaml     # token-vault-service allow rules
│   └── psp-connector-policy.yaml   # psp-connector egress-only rules
│
├── workloads/
│   ├── payment-authorization-api.yaml
│   ├── token-vault-service.yaml
│   └── psp-connector.yaml
│
├── services/
│   ├── payment-api-service.yaml
│   ├── token-vault-service.yaml
│   └── psp-connector.yaml
│
├── admission/
│   ├── kyverno-policies.yaml       # ClusterPolicy: 6 rules enforced at admission time
│   └── examples/
│       ├── README.md               # How to validate policies
│       ├── bad-pod-root.yaml       # Should be rejected by disallow-root
│       └── bad-pod-no-limits.yaml  # Should be rejected by require-resources
│
├── secrets/
│   ├── secret-store.yaml           # ClusterSecretStore: Vault backend via k8s auth
│   ├── external-secret.yaml        # ExternalSecret: PSP credentials synced from Vault
│   └── README.md                   # Secrets rotation and audit notes
│
├── tests/
│   └── attacker-pod.yaml           # Compliant test pod to validate zero-trust policies
│
├── docs/
│   ├── network-flow-diagram.md     # Full traffic topology and allowed/blocked paths
│   └── THREAT-ANALYSIS.md          # Threat model, design rationale, trade-offs, residual risks
│
└── README.md
```

## Services

Three services are deployed in the `cde-production` namespace:

**`payment-authorization-api`** is the entry point for payment requests from Yuno's orchestration layer. It calls `token-vault-service` to detokenize card data and `psp-connector` to forward the authorization request to the appropriate Payment Service Provider. It is the only service accessible from outside the namespace.

**`token-vault-service`** stores tokenized cardholder data and is the highest-risk component in the CDE. It accepts connections only from `payment-authorization-api` and the batch `reconciliation-job`. It has no permitted outbound connections.

**`psp-connector`** handles outbound HTTPS communication to external PSPs (Stripe, Adyen). It stores PSP API credentials injected at runtime via the External Secrets Operator. It accepts no inbound connections from any source.

## Security Controls

### Requirement 1 — Network Segmentation

A `default-deny-all` NetworkPolicy blocks all ingress and egress traffic in `cde-production` by default. Individual policies then carve out only the exact communication paths each service requires. No cross-namespace traffic is permitted unless explicitly granted via a `namespaceSelector` + `podSelector` conjunction. See `docs/network-flow-diagram.md` for the full topology.

### Requirement 2 — Pod Security (Restricted PSS)

All workloads conform to the Kubernetes `restricted` Pod Security Standard, enforced at the namespace level via the `pod-security.kubernetes.io/enforce: restricted` label. Per-container controls:

- Non-root UIDs (10001 / 10002 / 10003)
- `runAsNonRoot: true` at pod spec level
- `readOnlyRootFilesystem: true` (writable paths use named `emptyDir` volumes)
- `capabilities.drop: [ALL]`
- `allowPrivilegeEscalation: false`
- `seccompProfile: RuntimeDefault`
- CPU and memory `requests` and `limits` on every container

### Requirement 3 — Admission Policy Enforcement

Kyverno is deployed with a `ClusterPolicy` in `Enforce` mode. Six rules fire on every Pod admission request:

| Rule | What it blocks |
|---|---|
| `disallow-root` | Pods without `runAsNonRoot: true` at spec level |
| `disallow-root-container-override` | Containers explicitly setting `runAsUser: 0` |
| `disallow-privileged` | Containers with `privileged: true` |
| `disallow-hostpath` | Pods mounting `hostPath` volumes |
| `require-approved-registry` | Images not from `registry.yuno-internal.io/*` (covers `initContainers` and `ephemeralContainers`) |
| `require-resources` | Containers missing CPU or memory limits |

### Stretch Goal — Secrets Hardening

PSP API credentials are managed through HashiCorp Vault, accessed via the External Secrets Operator. The `ClusterSecretStore` authenticates to Vault using the Kubernetes auth method (service account JWT, no static tokens). Secrets are synced into `cde-production` every hour. Pods consume credentials at runtime via `envFrom.secretRef`. No secrets are stored in Git, CI logs, or container images. See `secrets/README.md` for rotation and audit details.

## Deployment Order

Apply resources in this order to avoid admission failures:

```bash
# 1. Install Kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

# 2. Namespace and quotas
kubectl apply -f base/

# 3. Admission policies (must be in place before workloads)
kubectl apply -f admission/kyverno-policies.yaml

# 4. Network policies
kubectl apply -f network-policies/

# 5. Services
kubectl apply -f services/

# 6. Secrets (requires ESO and Vault to be reachable)
kubectl apply -f secrets/secret-store.yaml
kubectl apply -f secrets/external-secret.yaml

# 7. Workloads
kubectl apply -f workloads/
```

## Validating Admission Policies

```bash
# Should be rejected: running as root
kubectl apply -f admission/examples/bad-pod-root.yaml

# Should be rejected: no resource limits
kubectl apply -f admission/examples/bad-pod-no-limits.yaml
```

## Testing Network Zero-Trust

The `tests/attacker-pod.yaml` deploys a compliant curl pod into `cde-production` simulating a compromised workload. Because the pod has no matching NetworkPolicy allow rules, all its outbound connection attempts should be silently dropped:

```bash
kubectl apply -f tests/attacker-pod.yaml

# Attempt to reach token-vault — should timeout
kubectl exec -it attacker -n cde-production -- curl -m 5 http://token-vault-service:8080

# Attempt to reach internet — should timeout
kubectl exec -it attacker -n cde-production -- curl -m 5 https://example.com
```

> **Note:** `tests/attacker-pod.yaml` uses `curlimages/curl`, which is an external image and will be rejected by the `require-approved-registry` Kyverno policy in a live cluster. To run this test, mirror the image to `registry.yuno-internal.io/curlimages/curl` first, or apply the pod in a namespace excluded from the ClusterPolicy.

## Further Reading

- `docs/network-flow-diagram.md` — complete traffic topology, allowed and blocked path tables, namespace isolation notes
- `docs/THREAT-ANALYSIS.md` — threat model, design decisions, trade-off analysis, residual risks, and future work
