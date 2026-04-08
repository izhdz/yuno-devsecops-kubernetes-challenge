# Network Flow Diagram — CDE Production

## Traffic Topology

```
                        ┌─────────────────────────────────────────────────────────┐
                        │                  cde-production namespace               │
                        │                                                         │
  [ingress-nginx ns]    │   ┌──────────────────────────────┐                     │
  ┌──────────────┐      │   │   payment-authorization-api   │                     │
  │ ingress-     │─────▶│   │   UID 10001 | port 8080       │                     │
  │ gateway      │      │   │   readOnly FS | caps: DROP ALL│                     │
  └──────────────┘      │   └────────────┬─────────┬────────┘                     │
                        │               │         │                              │
                        │    port 8080  │         │ port 8080                   │
                        │               ▼         ▼                              │
                        │   ┌──────────────┐  ┌──────────────┐                  │
                        │   │ token-vault  │  │ psp-connector│                  │
                        │   │ -service     │  │ UID 10003    │──────────────────▶ Internet
                        │   │ UID 10002    │  │ port 8080    │  HTTPS / 443 only  (Stripe, Adyen)
                        │   │ NO egress    │  │ NO inbound   │                  │
                        │   └──────┬───────┘  └──────────────┘                  │
                        │          │ ▲                                           │
                        │          │ │                                           │
                        │          └─┤                                           │
                        │   [batch-jobs ns]                                      │
                        │   reconciliation-job (inbound to token-vault only)     │
                        │                                                         │
                        │  ╔═══════════════════════════════════════════════╗     │
                        │  ║ default-deny-all: blocks ALL other traffic    ║     │
                        │  ╚═══════════════════════════════════════════════╝     │
                        └─────────────────────────────────────────────────────────┘

  [vault.yuno-internal.io]
  ┌────────────────────┐
  │ HashiCorp Vault    │◀── ESO ClusterSecretStore (k8s auth)
  │ secret/data/psp    │──▶ ExternalSecret → psp-connector-secrets (K8s Secret)
  └────────────────────┘    refreshInterval: 1h
```

## Allowed Communication Paths

| Source | Destination | Port | Protocol | Policy File |
|---|---|---|---|---|
| `ingress-gateway` (ingress-nginx ns) | `payment-authorization-api` | 8080 | TCP | `payment-api-policy.yaml` |
| `payment-authorization-api` | `token-vault-service` | 8080 | TCP | `payment-api-policy.yaml` |
| `payment-authorization-api` | `psp-connector` | 8080 | TCP | `payment-api-policy.yaml` |
| `psp-connector` | Internet (Stripe, Adyen, PSPs) | 443 | HTTPS/TCP | `psp-connector-policy.yaml` |
| `reconciliation-job` (batch-jobs ns) | `token-vault-service` | 8080 | TCP | `token-vault-policy.yaml` |
| ESO / Vault | `psp-connector` (secret sync) | — | Vault k8s auth | `secret-store.yaml` |

## Explicitly Blocked Paths

| Attempted Path | Blocked By |
|---|---|
| Any pod → any pod (default) | `default-deny-all` NetworkPolicy |
| `token-vault-service` → anywhere (outbound) | No egress rule exists; `default-deny-all` applies |
| `psp-connector` ← any inbound connection | `ingress: []` in `psp-connector-policy.yaml` |
| Any pod in `monitoring`, `dev`, or other namespaces → CDE services | No `namespaceSelector` grants cross-namespace access |
| `payment-authorization-api` → Internet (direct) | No egress rule to `ipBlock`; only pod-scoped egress allowed |
| Any pod with unlabelled or wrong labels → CDE services | `podSelector` matching required on all allow rules |

## NetworkPolicy Files

```
network-policies/
├── default-deny.yaml          # Blanket deny all ingress + egress in cde-production
├── payment-api-policy.yaml    # payment-api: ingress from ingress-gateway; egress to vault + psp
├── token-vault-policy.yaml    # token-vault: ingress from payment-api + reconciliation-job only
└── psp-connector-policy.yaml  # psp-connector: no inbound; egress HTTPS/443 only
```

## Namespace Isolation Notes

Kubernetes NetworkPolicies are namespace-scoped by default. A `podSelector`-only rule in `from` matches pods **within the same namespace only**. Cross-namespace traffic requires an explicit `namespaceSelector` conjunction. This is applied in two places:

- `payment-api-policy.yaml` uses `namespaceSelector: kubernetes.io/metadata.name: ingress-nginx` combined with `podSelector: app: ingress-gateway` to allow only the specific ingress controller pod in its own namespace. The two selectors are siblings under the same `from` list item (AND logic), not separate items (OR logic).
- `token-vault-policy.yaml` uses `namespaceSelector: kubernetes.io/metadata.name: batch-jobs` combined with `podSelector: app: reconciliation-job` for the batch reconciliation workload.

Any pod in `dev`, `monitoring`, `staging`, or any other namespace attempting to reach a service in `cde-production` will be silently dropped.
