# CRCD Kubernetes Configuration

Production manifests for deploying administrative applications on the Pitt CRCD Kubernetes cluster.

## How This Repository Works

The CRCD Kubernetes cluster leverages a GitOps deployment model powered by [ArgoCD](https://argo-cd.readthedocs.io/).
Rather than deploying applications manually, manifests are loaded from this repository
and automatically reconciled against the live cluster. If a resource drifts from what
is defined here, ArgoCD corrects it.

## Repository Structure

This repository provides manifests for three groups of applications.

- **argocd/**: The ArgoCD application.
- **apps/**: Cluster-wide administrative services (one subdirectory per service)
- **tenants/**: Configures customer/tenant access to the cluster (one file per tenant)

### ArgoCD (`argocd/`)

ArgoCD manages itself via a self-referencing `Application` resource that points at
the `argocd/` directory in this repository. That directory contains a Kustomization
file which renders the ArgoCD Helm chart along with customized resource manifests used
to configure the deployed application.

### Administrative Services (`apps/`)

Administrative applications are used to monitor the production cluster and enforce
general operational policies. This includes applications required by the Pitt Digital
and Pitt Security teams. Changes to any administrative apps should be confirmed with
a Pitt Digital administrator before being pushed to production.

### Tenant Workloads (`tenants/`)

Tenants manage their application code and Kubernetes manifests in their own git
repositories. This repository owns the platform-side configuration that controls
how those workloads run on the cluster.

A tenant's workload will not be deployed, and will not have cluster access, without
having a corresponding ArgoCD configuration in this repository. When onboarding a new
tenant, the necessary ArgoCD resources must be added here first.

**What lives in this repo:**

- Namespace declarations and resource quotas
- `AppProject` definitions used to enforce per-tennant security, governance, and RBAC policies
- `Application` or `ApplicationSet` definitions configuring tenant repository deployment

**What lives in the tenant repo:**

- Application source code
- Application manifests, Helm charts, and configs
- Runtime configuration (e.g., env vars)
- CI/CD pipelines

## Common Tasks

### Adding a New Administrative Service

Create a subdirectory under `apps/` containing the relevant manifests and merge into
`main`. All admin services should be installed into a namespace with the `admin-` prefix.
ArgoCD will provision the service automatically on its next sync.

### Onboarding a New Tenant

The tenant's own repository is responsible for the manifests that run their workload.
This repository is responsible for granting it a place to run.
To deploy a new tenant application:

1. Create a new file under `tenants/` (e.g. `tenants/<tenant-name>.yaml`).
2. Add an `AppProject` defining which source repositories and namespaces the tenant
   is permitted to use.
3. Add an `Application` or `ApplicationSet` pointing at the tenant's own repository.
4. Merge the changes into `main` via a pull request. ArgoCD will provision everything
   on the next sync.
