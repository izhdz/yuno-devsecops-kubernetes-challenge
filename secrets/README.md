Secrets are rotated in Vault.

External Secrets Operator syncs updates every 1h.

Pods receive updated secrets via:
- restart (rolling deployment), OR
- sidecar reloader (future improvement)
- All secret access is logged in Vault audit logs
- Kubernetes only sees ephemeral synced secrets
- No secrets stored in Git or CI logs

In this implementation, secrets still exist in Kubernetes as native Secrets (base64 encoded).

Mitigation:
- Enable encryption at rest (KMS)
- Restrict RBAC access to secrets
- Prefer short TTL secrets from Vault