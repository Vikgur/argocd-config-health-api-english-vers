# Table of Contents

- [About the Project](#about-the-project)  
- [Architecture and Configuration](#architecture-and-configuration)  
- [How to Apply and Manage](#how-to-apply-and-manage)  
- [Implemented DevSecOps Practices](#implemented-devsecops-practices)  
  - [AppProjects: Environment Isolation and Access Control](#appprojects-environment-isolation-and-access-control)  
  - [SSO: Purpose, Preparation, Implementation](#sso-purpose-preparation-implementation)  
    - [Purpose](#purpose)  
    - [Configuration Steps (GitHub)](#configuration-steps-github)  
    - [Support for Two Authentication Methods](#support-for-two-authentication-methods)  
  - [RBAC: Access Segmentation](#rbac-access-segmentation)  
    - [Important Condition](#important-condition)  
  - [Telegram Notifications Setup](#telegram-notifications-setup)  
    - [Notification Logic](#notification-logic)  
    - [How to Get the Token and Chat ID](#how-to-get-the-token-and-chat-id)
  - [Linting and Validation](#linting-and-validation)

---

# About the Project

## About the Project

This repository is the **core GitOps configuration of Argo CD** for the [`health-api`](https://github.com/vikgur/health-api-for-microservice-stack-english-vers) web application. It establishes the foundation of platform management: defining **AppProjects**, configuring **SSO/local authentication**, setting up **RBAC policies**, connecting required **Git repositories**, and specifying controller parameters along with custom health checks. Additionally, it integrates a **Telegram notification system** that provides a full feedback cycle — from messages about successful deployments (**on-deployed**) to alerts on degraded application states (**on-health-degraded**).  

The key architectural principle is that **Argo CD does not manage this repository — this repository manages Argo CD**, defining its configuration, security, and behavior. Configuration is applied declaratively through **Kustomize** (`kustomize build` + `kubectl apply`), embedded in the infrastructure pipeline powered by Ansible: [`ansible-gitops-bootstrap-health-api`](https://github.com/vikgur/ansible-gitops-bootstrap-health-api-english-vers).  

Through the **[`apps/platform-apps.yaml`](apps/platform-apps.yaml)** object, this repository also connects the infrastructure stack [`gitops-argocd-platform-health-api`](https://github.com/vikgur/gitops-argocd-platform-health-api-english-vers) following the *App of Apps* pattern. This ensures that core system components (ingress-nginx, cert-manager, external-secrets, etc.) are provisioned and maintained by Argo CD automatically, without direct Ansible involvement.

---

# Architecture and Configuration

* **System applications integration** (`apps/platform-apps.yaml`)  
  * Implements the link to the [`gitops-argocd-platform-health-api`](https://github.com/vikgur/gitops-argocd-platform-health-api-english-vers) repository, which defines all infrastructure components (ingress-nginx, cert-manager, external-secrets, monitoring, etc.).  
  * This ensures a clear separation: Argo CD configuration is stored here, while the platform installation is maintained in a dedicated repository.  

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

# Implemented DevSecOps Practices
The repository implements a secure, declarative access control configuration for Argo CD.

## AppProjects: Environment Isolation and Access Control

Access restrictions are defined in the `argocd/projects/` directory:

- strict binding to specific namespaces and Git repositories for each environment (`stage`, `prod`)
- orphaned resource warnings enabled (`orphanedResources.warn`)
- a sync window is configured for `prod` — deployments are only allowed during working hours

## SSO: Purpose, Preparation, Implementation

### Purpose

**Goal:** Secure centralized login to Argo CD via GitHub, without manual logins.

* User logs in via GitHub OAuth
* Argo CD receives `email`, `username`, `groups`
* Groups (`g:devops`, `g:qa`) control access via `argocd-rbac-cm.yaml`

### Setup Steps (GitHub)

1. **Register an OAuth App in GitHub:**

   * Go to: `GitHub → Settings → Developer settings → OAuth Apps`
   * Click **New OAuth App**:

     * Application Name: `Argo CD SSO`
     * Homepage URL: `https://argocd.health.gurko.ru`
     * Authorization callback URL for OIDC:
       `https://argocd.health.gurko.ru/auth/callback`
     * Authorization callback URL for DEX:
       `https://argocd.health.gurko.ru/api/dex/callback`

2. **Copy:**

   * `Client ID`
   * `Client Secret`

3. **Paste into [ansible-gitops-bootstrap-health-api](https://github.com/Vikgur/ansible-gitops-bootstrap-health-api-english-vers/) in [ansible/group\_vars/master.yaml](https://github.com/Vikgur/ansible-gitops-bootstrap-health-api-english-vers/-/blob/main/ansible/group_vars/master.yaml):**

   ```yaml
   github_oauth_client_id: YOUR_CLIENT_ID
   github_oauth_client_secret: YOUR_CLIENT_SECRET
   argocd_sso_mode: oidc # or dex
   ```

### Support for Two Authentication Methods

The repository contains two independent authentication approaches for Argo CD:

* **OIDC directly via GitHub (chosen as the main one)** — used in production, configured in `argocd-cm.yaml`, without additional components. In the linked repository [ansible-gitops-bootstrap-health-api](https://github.com/Vikgur/ansible-gitops-bootstrap-health-api-english-vers) in [ansible/group_vars/master.yaml](https://github.com/Vikgur/ansible-gitops-bootstrap-health-api-english-vers/-/blob/main/ansible/group_vars/master.yaml), `argocd_namespace: argocd` is chosen.
* **Dex + GitHub OAuth** — an additional demonstration option for portfolio purposes, located in `argocd-cm-dex.yaml`.

Both files can be applied manually via `kubectl apply`, depending on the required configuration.

The active option is specified via the file `argocd/cm/argocd-cm-*.yaml`, others are commented out in `kustomization.yaml`

#### Why OIDC was chosen:

* Simplifies architecture: fewer components = fewer failure points.
* Configured directly in `argocd-cm.yaml` via `oidc.config`.
* Better suited for cloud CI/CD systems (GitHub, github, Okta).
* Used in production clusters of large companies.

Dex is a more “architectural” method, used if a company has multiple providers (GitHub, github, LDAP, etc.). Dex is kept in the project **to demonstrate an alternative option**.

## RBAC: Access Control

The file `argocd/cm/argocd-rbac-cm.yaml` defines roles and access permissions in Argo CD:

- **Role `admin`** — full access to all applications and projects (`stage` and `prod`)
- **Role `stage-admin`** — access limited to the `stage` environment, no permissions for `prod`

Roles are assigned to user groups:

- `g:devops` → assigned the `admin` role
- `g:qa` → assigned the `stage-admin` role

This setup restricts access to `prod`, allows QA to work in `stage`, and prevents unintended actions outside a user's scope. All permissions are managed declaratively through GitOps.

### Important Condition

`g:devops`, `g:qa` must be GitHub Teams when using organizations.  
If logging in with individual users — use `login:<user>` instead of `g:...`.

## Telegram Notifications Setup

### Notification Logic

This repository includes a notification system integrated with Telegram:

* **on-deployed** — sent after a successful application deployment
* **on-health-degraded** — sent when application status becomes Degraded

Notification logic is defined in `argocd/notifications/triggers.yaml`, templates in `argocd/notifications/templates.yaml`, and the Telegram channel is configured via `argocd/notifications/cm.yaml` and `argocd/notifications/secret.yaml`.

> The values `telegram_token` and `telegram_chat_id` are provided via the `argocd/notifications/secret.yaml` file.

### How to Get the Token and Chat ID

**1. `TELEGRAM_TOKEN`**
Telegram bot token:

* Create a bot via `@BotFather` using `/newbot`
* Example name: `argocd_notifications_bot`
* You will receive a token like `123456789:AAFx8Z...`

**2. `TELEGRAM_CHAT_ID`**
The chat ID where messages will be sent:

* Add the bot to a group or send it a direct message

* Run:

  ```bash
  curl "https://api.telegram.org/bot<TELEGRAM_TOKEN>/getUpdates"
  ```

* In the response, find:

  ```json
  "chat": {
    "id": -123456789
  }
  ```

* Use this value as your `TELEGRAM_CHAT_ID`

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
