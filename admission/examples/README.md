# Admission Policy Validation Examples

These manifests are intentionally insecure and should be rejected by Kyverno policies.
Please refer to them before adding your own manifests for CDE Staging AND/OR CDE Production

## Test

kubectl apply -f bad-pod-root.yaml

Expected:
- Rejected by `disallow-root` policy

kubectl apply -f bad-pod-no-limits.yaml

Expected:
- Rejected by `require-resources` policy