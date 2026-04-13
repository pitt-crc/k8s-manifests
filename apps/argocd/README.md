# ArgoCD

Deploys Argo CD onto the cluster using the official Helm chart, rendered via Kustomize.
Once synced, ArgoCD manages its own installation — changes to `values.yaml` or `kustomization.yaml` are applied 
automatically on the next reconciliation.

## Deployment

Deployed via Kustomize using a `helmCharts` entry in `kustomization.yaml`.
Chart version and release name are pinned in `kustomization.yaml`.
Instance configuration is declared in `values.yaml`.

TLS is handled by cert-manager using a `Certificate` resource.
The resulting secret is referenced directly in the ArgoCD server configuration.
