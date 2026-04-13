# k8s-manifests

Production Kubernetes manifests for the Pitt CRCD cluster.

This repository is the source of truth for core management infrastructure.
It is responsible for managing cluster stateincluding GitOps tooling and cluster-wide
services.
It also contains the ArgoCD configuration that governs how tenant workloads are deployed.

## How This Repository Works

The cluster uses a GitOps model powered by [ArgoCD](https://argo-cd.readthedocs.io/).
Rather than applying manifests manually, manifests are loaded from this repository and
automatically reconciled against the live cluster. If a resource drifts from what
is defined here, ArgoCD corrects it.

### Repository Structure

This repository uses an App-of-Apps pattern. Two root-level manifests bootstrap
the entire system:

- `apps/project.yaml` defines an ArgoCD `AppProject` that scopes all admin
  applications to this repository and the `admin-*` namespaces.
- `apps/applicationset.yaml` defines an `ApplicationSet` that watches the `apps/`
  directory and automatically creates a deployment for each subdirectory it finds.

Each subdirectory under `apps/` represents a single platform service such as ArgoCD
itself, ingress, or cert-manager. Adding a directory is enough to deploy a new
service with no manual ArgoCD configuration required.

### Tenant Workloads

Tenants manage their application code and Kubernetes manifests in their own repositories.
This repository owns the platform-side configuration that controls how those workloads run on the cluster:

**What lives here:**

- `AppProject` definitions (what namespace and repo access rules)
- `Application` or `ApplicationSet` definitions
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

## Adding a New Administrative Service

Deployment manifests for cluster-wide admin services are stored directly in this repository.
Create a subdirectory under `apps/` containing the relevant service manifests, Kustomization files, or Helm values.
Merge the changes into `main` via a pull request and ArgoCD will provision the application on its next sync.

## Onboarding a New Tenant

The tenant's own repository is responsible for the manifests that run their workload.
This repository is responsible for granting it a place to run.

1. Create a subdirectory under `apps/` for the tenant (e.g. `apps/<tenant-name>/`).
2. Add an `AppProject` defining which source repositories and namespaces the tenant is permitted to use.
3. Add an `Application` or `ApplicationSet` pointing at the tenant's own repository.
4. Merge the changes into `main` via a pull request. ArgoCD will provision everything on the next sync.
