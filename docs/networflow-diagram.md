## Network Flow Policies Diagram

Ingress → payment-api → token-vault
                    → psp-connector → Internet (HTTPS only)

token-vault ← ONLY payment-api + batch job

## NO cross-namespace traffic
## NO inbound to psp-connector