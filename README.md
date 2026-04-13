# k8s-manifests

Production manifests for core infrastructure on the Pitt CRCD Kubernetes cluster.
This includes GitOps tooling, certificate management, logging, and other cluster-wide services.

## Repository Structure

This repository uses the ArgoCD App-of-Apps pattern.
Two root-level manifests define the pattern:

- `apps/project.yaml` — creates an ArgoCD `AppProject` scoping all applications to this repository and the `admin-*`
  namespaces
- `apps/applicationset.yaml` — defines an `ApplicationSet` that watches the `apps/` directory and automatically creates
  an ArgoCD `Application` for each subdirectory it finds

Each subdirectory under `apps/` represents a single cluster service, deployed into a dedicated `admin-<service-name>`
namespace.

## Adding a New Admin Application

Admin applications are cluster-level services managed by this repository.
To add one, create a subdirectory under `apps/` containing the relevant manifests, Kustomization, or Helm chart values,
then commit and push to `main`.
The ApplicationSet will detect the new directory on the next sync and deploy it automatically.

## Managing Tenant Workloads

Tenant application code and Kubernetes manifests live in their own repositories.
This repository owns the ArgoCD-side configuration that governs how those workloads are deployed:
`AppProjects`, `Applications`, and `ApplicationSets` for tenant namespaces belong here.

Changes to tenant deployment configuration — such as adding a new project, onboarding a new application, or modifying
sync policy — should be made in this repository and applied through ArgoCD.
