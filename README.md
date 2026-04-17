# k8s-manifests

Production Kubernetes manifests for the Pitt CRCD cluster.

This repository is the source of truth for core management infrastructure.
It is responsible for managing cluster state including GitOps tooling and cluster-wide
services. It also contains the ArgoCD configuration that governs how tenant workloads
are deployed.

## How This Repository Works

The cluster uses a GitOps model powered by [ArgoCD](https://argo-cd.readthedocs.io/).
Rather than applying manifests manually, manifests are loaded from this repository and
automatically reconciled against the live cluster. If a resource drifts from what
is defined here, ArgoCD corrects it.

### Repository Structure

This repository uses an App-of-Apps pattern. Two root-level manifests bootstrap
the entire system:

- **argocd/**: ArgoCD self-management (Helm chart + configuration)
- **apps/**: Cluster-wide admin services (one subdirectory per service)
- **tenants/**: Tenant ArgoCD configuration (one file per tenant)

ArgoCD manages itself via a self-referencing `Application` that points at the `argocd/`
directory. That directory contains a Kustomization which renders the ArgoCD Helm chart
alongside the ArgoCD resources that define the rest of the system.

Two additional Applications are bootstrapped from `argocd/`:

- `argocd/application.yaml` — the self-manage Application, which reconciles the
  `argocd/` directory itself.
- `argocd/tenants.yaml` — a plain Application that watches the `tenants/` directory
  and automatically deploys any manifest files it finds there.

The `tenants/admin.yaml` file defines the `admin-apps` AppProject and an
ApplicationSet that watches `apps/*`, automatically creating a deployment for each
subdirectory it finds there.

### Adding a New Administrative Service

Create a subdirectory under `apps/` containing the relevant service manifests,
Kustomization files, or Helm values. Merge the changes into `main` via a pull request
and ArgoCD will provision the application on its next sync. No manual ArgoCD
configuration is required.

### Tenant Workloads

Tenants manage their application code and Kubernetes manifests in their own
repositories. This repository owns the platform-side configuration that controls
how those workloads run on the cluster.

**What lives here (in `tenants/`):**

- `AppProject` definitions (namespace and repository access rules)
- `Application` or `ApplicationSet` definitions pointing at tenant repositories
- Namespace declarations and resource quotas
- RBAC for tenant teams

**What lives in the tenant repo:**

- Application source code
- Kubernetes manifests and Helm values
- Runtime configuration (secrets, env vars)
- CI/CD pipelines

A tenant's workload will not be deployed, and will not have cluster access, without
the corresponding ArgoCD configuration in this repository. When onboarding a new
tenant, the necessary ArgoCD resources must be added here first.

## Bootstrapping the Cluster

ArgoCD is installed once manually via Helm. After that, it manages itself and
everything else through this repository. See `argocd/BOOTSTRAP.md` for the full
procedure.

## Adding a New Administrative Service

Create a subdirectory under `apps/` containing the relevant manifests and merge into
`main`. ArgoCD will provision the service automatically on its next sync.

## Onboarding a New Tenant

The tenant's own repository is responsible for the manifests that run their workload.
This repository is responsible for granting it a place to run.

1. Create a new file under `tenants/` (e.g. `tenants/<tenant-name>.yaml`).
2. Add an `AppProject` defining which source repositories and namespaces the tenant
   is permitted to use.
3. Add an `Application` or `ApplicationSet` pointing at the tenant's own repository.
4. Merge the changes into `main` via a pull request. ArgoCD will provision everything
   on the next sync.