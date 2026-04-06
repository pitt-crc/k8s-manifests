# cert-manager Sectigo ACME ClusterIssuer

Creates the Sectigo EAB secret and ClusterIssuer for automatic TLS
certificate provisioning via ACME HTTP-01 challenges.

## Prerequisites

Install cert-manager from upstream (run once):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl -n cert-manager rollout status deployment/cert-manager-webhook
```

## Placeholders

Replace in `cert-manager.yaml` before deploying:

- `<eab-key-id>` - Sectigo External Account Binding key ID
- `<eab-hmac-key>` - Sectigo EAB HMAC key (plain text)

## Deploy

```bash
kubectl apply -f cert-manager.yaml
```

## Verify

```bash
kubectl get clusterissuer sectigo-issuer
```
