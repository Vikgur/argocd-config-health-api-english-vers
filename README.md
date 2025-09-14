# Table of Contents

- [About the Project](#about-the-project)  
- [Architecture and Configuration](#architecture-and-configuration)  
- [How to Apply and Manage](#how-to-apply-and-manage)  
- [Implemented DevSecOps Practices](#implemented-devsecops-practices)  
  - [RBAC: Access Control](#rbac-access-control)  
  - [AppProjects: Environment Isolation and Access Control](#appprojects-environment-isolation-and-access-control)  
  - [Linting and Validation](#linting-and-validation)

---

# About the Project

This repository serves as the core GitOps configuration for Argo CD in the [`health-api`](https://github.com/vikgur/health-api-for-microservice-stack-english-vers) web application.  
It defines all critical Argo CD components — **RBAC policy**, **AppProjects**, Git **repository integrations**, controller settings, and custom health checks.

Argo CD does not manage this repository — instead, **this repository manages Argo CD**.

The configuration is applied declaratively via `kustomize build` and `kubectl apply` as part of an Ansible-based infrastructure pipeline.

Deployment and automation are handled through the dedicated Ansible project:  
[`ansible-gitops-bootstrap-health-api`](https://github.com/vikgur/ansible-gitops-bootstrap-health-api-english-vers)

---

# Architecture and Configuration

* **AppProjects** (`argocd/projects/`)

  * `project-stage.yaml` — defines the `stage` environment, limited to the `health-api-stage` namespace and specific Git repositories.
  * `project-prod.yaml` — defines the `prod` environment, limited to the `health-api` namespace and enforces a sync window (deployments allowed only during working hours).

* **Git Repositories** (`argocd/repos/`)

  * `repo-gitops-apps.yaml` — grants access to the `gitops-apps-health-api` repository containing Argo CD Applications.
  * `repo-helm-charts.yaml` — grants access to the `helm-blue-green-canary-gitops-health-api` repository containing Helm charts.

* **Argo CD Controller Configuration** (`argocd/cm/argocd-cm.yaml`)

  * Sets the `application.instanceLabelKey` for correct resource association.
  * Includes custom health checks for CRDs (e.g., Rollout).
  * Configures reconciliation timeouts.

* **RBAC Configuration** (`argocd/cm/argocd-rbac-cm.yaml`)

  * Defines roles: `admin` and `stage-admin`.
  * Maps access policies to user groups (`g:devops`, `g:qa`).

* **argocd/kustomization.yaml**

  * Centralized entry point for applying the full configuration using:

```bash
kustomize build . | kubectl apply -f -
```

---

# How to Apply and Manage

```bash
# Install kustomize if not already installed
sudo snap install kustomize

# Apply the full Argo CD configuration from this repository
kustomize build . | kubectl apply -f -
```

---

# Внедренные DevSecOps практики
The repository implements a secure, declarative access control configuration for Argo CD.

## RBAC: Access Control

The file `argocd/cm/argocd-rbac-cm.yaml` defines roles and access permissions in Argo CD:

- **Role `admin`** — full access to all applications and projects (`stage` and `prod`)
- **Role `stage-admin`** — access limited to the `stage` environment, no permissions for `prod`

Roles are assigned to user groups:

- `g:devops` → assigned the `admin` role
- `g:qa` → assigned the `stage-admin` role

This setup restricts access to `prod`, allows QA to work in `stage`, and prevents unintended actions outside a user's scope. All permissions are managed declaratively through GitOps.

## AppProjects: Environment Isolation and Access Control

Access restrictions are defined in the `argocd/projects/` directory:

- strict binding to specific namespaces and Git repositories for each environment (`stage`, `prod`)
- orphaned resource warnings enabled (`orphanedResources.warn`)
- a sync window is configured for `prod` — deployments are only allowed during working hours

## Linting and Validation

Following the GitOps approach, the Argo CD configuration is written as declarative code.  
To ensure structural integrity, readability, and compliance with Kubernetes specifications, basic linters are integrated into the project.  
The checks are minimal but essential — they guarantee that any configuration changes are automatically validated before being applied.

The repository includes pre-commit hooks to validate YAML and Kubernetes manifests:

- **yamllint** — validates YAML structure and syntax  
- **ct lint** — validates Kubernetes manifests (schema correctness, key usage, etc.)

The hook configuration is defined in [.pre-commit-config.yaml](./.pre-commit-config.yaml) and runs automatically on each commit.

Manual run:

```bash
pre-commit run --all-files
```
