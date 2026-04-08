# nginx

Simple nginx deployment with its own LoadBalancer IP. Serves a hello
world page showing the deployment name, node, and pod.

## Deploy

```bash
kubectl apply -f nginx.yaml
```

## Verify

```bash
kubectl get pods -n nginx
curl http://<your-url>
```
