# Fluent Bit

Log collector deployed as a DaemonSet. Collects container logs and
systemd journal entries, then forwards them to Cribl via syslog
(RFC 5424, UDP).

## Deploy

```bash
kubectl apply -f fluent-bit.yaml
```

## Verify

```bash
kubectl get pods -n fluent-bit
kubectl logs -n fluent-bit -l k8s-app=fluent-bit-logging --tail=20
```
