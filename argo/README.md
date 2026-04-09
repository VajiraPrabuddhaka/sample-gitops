# GitOps with Argo CD

This tutorial walks through setting up a complete GitOps workflow using OpenChoreo and Argo CD with this sample-gitops repository. You will configure Argo CD for resource synchronization, build and deploy a multi-component application using OpenChoreo Workflows, and promote components across environments.

**Learning objectives:**

- Argo CD setup for OpenChoreo resource synchronization
- App of Apps pattern for managing multiple Argo CD Applications
- GitOps repository structure (platform vs. namespace organization)
- Component building and deployment automation via OpenChoreo Workflows
- ComponentReleases and ReleaseBindings for environment promotion
- Development to staging pipeline execution (bulk component promotion)

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Fork and Clone the Sample Repository](#step-1-fork-and-clone-the-sample-repository)
- [Step 2: Update Repository URLs](#step-2-update-repository-urls)
- [Step 3: Create Git Secrets](#step-3-create-git-secrets)
- [Step 4: Deploy the Argo CD Root Application](#step-4-deploy-the-argo-cd-root-application)
- [Step 5: Verify Platform Resources](#step-5-verify-platform-resources)
- [Step 6: Build and Deploy the Doclet Application](#step-6-build-and-deploy-the-doclet-application)
- [Step 7: Promote to Staging](#step-7-promote-to-staging)
- [Step 8: Environment-Specific Overrides](#step-8-environment-specific-overrides)
- [Clean Up](#clean-up)

---

## Prerequisites

### Install OpenChoreo

Follow the official documentation: [Try it out on k3d locally](https://openchoreo.dev/docs/getting-started/try-it-out/on-k3d-locally/)

> [!WARNING]
> Do **not** install the OpenChoreo default resources. Only create the **default clusterdataplane** and **clusterworkflowplane**.

### Required Tools

- `kubectl` configured for cluster access
- `git` CLI installed
- A GitHub account for repository forking

### Install Argo CD

Install Argo CD into your cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD to be ready:

```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s
```

Optionally, install the Argo CD CLI for easier management:

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```

> [!NOTE]
> This tutorial assumes a k3d local setup. Adjust accordingly for other cluster types.

---

## Step 1: Fork and Clone the Sample Repository

1. Navigate to the [sample-gitops](https://github.com/openchoreo/sample-gitops) GitHub repository.
2. Click **Fork** in the top-right corner.
3. Clone your forked repository locally:

```bash
git clone https://github.com/<your-github-username>/sample-gitops.git
cd sample-gitops
```

---

## Step 2: Update Repository URLs

Argo CD and the build workflows need to know the location of your forked repository.

### 2.1 Update Argo CD Application manifests

Update the `spec.source.repoURL` field in the following files to point to your fork:

- [`argo/root-application.yaml`](./root-application.yaml)
- [`argo/apps/namespaces.yaml`](./apps/namespaces.yaml)
- [`argo/apps/platform-shared.yaml`](./apps/platform-shared.yaml)
- [`argo/apps/oc-demo-platform.yaml`](./apps/oc-demo-platform.yaml)
- [`argo/apps/oc-demo-projects.yaml`](./apps/oc-demo-projects.yaml)

Replace the `repoURL` in each file:

```yaml
spec:
  source:
    repoURL: https://github.com/<your-github-username>/sample-gitops
```

### 2.2 Update Workflow files

Update the `gitops-repo-url` parameter in each of the following workflow files to point to your fork:

- [`namespaces/default/platform/workflows/docker-with-gitops.yaml`](../namespaces/default/platform/workflows/docker-with-gitops-release.yaml)
- [`namespaces/default/platform/workflows/google-cloud-buildpacks-gitops-release.yaml`](../namespaces/default/platform/workflows/google-cloud-buildpacks-gitops-release.yaml)
- [`namespaces/default/platform/workflows/react-gitops-release.yaml`](../namespaces/default/platform/workflows/react-gitops-release.yaml)

Commit and push all URL changes to your forked repository.

### 2.3 Generate a GitHub PAT

Generate a [GitHub Personal Access Token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) with read/write access to your forked repository.

> [!NOTE]
> If your repository is **private**, you will also need to configure an Argo CD repository credential. See the [Argo CD private repositories guide](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/).

---

## Step 3: Create Git Secrets

[OpenChoreo Workflows](../namespaces/default/platform/workflows/README.md) need access to your repositories for cloning source code and pushing GitOps manifests. Store your GitHub PAT in the OpenBao secret store:

```bash
# Secret for cloning source repositories
kubectl exec -n openbao openbao-0 -- bao kv put secret/git-token git-token=<your_github_pat>

# Secret for pushing to and creating PRs in the GitOps repository
kubectl exec -n openbao openbao-0 -- bao kv put secret/gitops-token git-token=<your_github_pat>
```

Replace `<your_github_pat>` with your actual token.

---

## Step 4: Deploy the Argo CD Root Application

This setup uses the [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern. A single root Application points to the `argo/apps/` directory, which contains child Application manifests that each sync a different part of the repository.

Apply the root Application:

```bash
kubectl apply -f argo/root-application.yaml
```

This creates the root Application (`sample-gitops`), which automatically creates four child Applications:

| Application | Path | Sync Wave | Purpose |
|---|---|---|---|
| **namespaces** | `namespaces/` | 0 | Namespace definitions |
| **platform-shared** | `platform-shared/` | 0 | Cluster-scoped resources (ClusterWorkflowTemplates) |
| **oc-demo-platform** | `namespaces/default/platform/` | 1 | Platform resources (Environments, ComponentTypes, Workflows) |
| **oc-demo-projects** | `namespaces/default/projects/` | 2 | Application resources (Projects, Components) |

Argo CD uses [sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) to enforce the correct apply order:

```mermaid
graph TD
    A["Root Application\n(sample-gitops)"] --> B["Application\n(namespaces)\nwave: 0"]
    A --> C["Application\n(platform-shared)\nwave: 0"]
    A --> D["Application\n(oc-demo-platform)\nwave: 1"]
    A --> E["Application\n(oc-demo-projects)\nwave: 2"]

    style A fill:#4a90d9,color:#fff
    style B fill:#6ab04c,color:#fff
    style C fill:#6ab04c,color:#fff
    style D fill:#f0932b,color:#fff
    style E fill:#eb4d4b,color:#fff
```

This ensures that namespaces and cluster-scoped resources are created first (wave 0), followed by platform resources (wave 1), and finally the application resources (wave 2).

Verify the Applications are created and synced:

```bash
kubectl get applications -n argocd
```

You should see all five Applications (the root plus four children) with a status of `Synced` and `Healthy`.

To check the sync status of a specific Application:

```bash
kubectl get application <application-name> -n argocd -o jsonpath='{.status.sync.status}'
```

To trigger an immediate sync:

```bash
# Using kubectl
kubectl patch application sample-gitops -n argocd --type merge -p '{"operation":{"sync":{"prune":true}}}'

# Or using the Argo CD CLI
argocd app sync sample-gitops
```

---

## Step 5: Verify Platform Resources

Within 1-2 minutes, Argo CD will sync the platform directory. Verify the platform resources:

```bash
kubectl get environments              # development, staging, production
kubectl get deploymentpipelines       # standard
kubectl get componenttypes            # deployment/service, deployment/web-application, deployment/database, deployment/message-broker
```

The `standard` DeploymentPipeline defines the promotion sequence: **development** -> **staging** -> **production**.

---

## Step 6: Build and Deploy the Doclet Application

The sample uses the **Doclet** application. Which is a multi-component system with two backend services and a frontend. The `postgres` DB and `nats` broker is deployed by default. Other components will be deployed by triggering OpenChoreo Workflow runs that build container images and create pull requests in your GitOps repository.

### 6.1 Document Service

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: document-svc-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "document-svc"
spec:
  workflow:
    name: docker-gitops-release
    kind: Workflow
    parameters:
      componentName: document-svc
      projectName: doclet
      docker:
        context: /project-doclet-app/service-go-document
        filePath: /project-doclet-app/service-go-document/Dockerfile
      repository:
        appPath: /project-doclet-app/service-go-document
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```

### 6.2 Collaboration Service

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: collab-svc-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "collab-svc"
spec:
  workflow:
    kind: Workflow
    name: docker-gitops-release
    parameters:
      componentName: collab-svc
      projectName: doclet
      docker:
        context: /project-doclet-app/service-go-collab
        filePath: /project-doclet-app/service-go-collab/Dockerfile
      repository:
        appPath: /project-doclet-app/service-go-collab
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```

### 6.3 Frontend

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: frontend-workflow-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "frontend"
spec:
  workflow:
    kind: Workflow
    name: docker-gitops-release
    parameters:
      componentName: frontend
      projectName: doclet
      docker:
        context: /project-doclet-app/webapp-react-frontend
        filePath: /project-doclet-app/webapp-react-frontend/Dockerfile
      repository:
        appPath: /project-doclet-app/webapp-react-frontend
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```

> [!NOTE]
> The source code for the Doclet application is available at [openchoreo/sample-workloads](https://github.com/openchoreo/sample-workloads/tree/main/project-doclet-app).


### 6.4 Merge the Pull Requests

Once all three workflows complete, **3 pull requests** will be created in your forked GitOps repository. One for each component. Each PR contains `Workload`, `ComponentRelease`, and `ReleaseBinding` manifests targeting the **development** environment.

Review and merge all pull requests, then wait for Argo CD to sync and deploy the components.

### 6.5 Verify the Deployment

```bash
kubectl get releasebindings
kubectl get deployments -A
kubectl get pods -A
```

---

## Step 7: Promote to Staging

After validating in the development environment, promote the entire **Doclet** project to staging using the bulk release workflow:

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: bulk-release-manual-01
  namespace: default
spec:
  workflow:
    kind: Workflow
    name: bulk-gitops-release
    parameters:
      scope:
        all: false
        projectName: "doclet"
      gitops:
        repositoryUrl: "https://github.com/<your-github-username>/sample-gitops"
        branch: "main"
        targetEnvironment: "staging"
        deploymentPipeline: "standard"
EOF
```

Replace `<your-github-username>` with your GitHub username. Once the workflow completes, a pull request will be created to promote all Doclet components from development to staging in a single operation.

Merge the pull request and wait for Argo CD to sync, then verify:

```bash
kubectl get releasebindings
```

You should see ReleaseBindings for both the **development** and **staging** environments. The same ComponentReleases are deployed to multiple environments, demonstrating the ReleaseBinding model: **one immutable release, multiple environments**.

---

## Step 8: Environment-Specific Overrides

Different environments often require different configurations. For example, staging might use different database endpoints or log levels than development. In OpenChoreo, the **ReleaseBinding** defines environment-specific overrides.

### Override types

| Override Type                     | Purpose                                                               |
|-----------------------------------|-----------------------------------------------------------------------|
| `componentTypeEnvironmentConfigs` | Override ComponentType parameters for specific environments           |
| `traitEnvironmentConfigs`         | Override Trait configurations for specific environments               |
| `workloadOverrides`               | Override workload-level settings (environment variables, file mounts) |

> [!IMPORTANT]
> Environment-specific overrides are not supported by the included GitOps workflows. You must manually add override configurations to the ReleaseBinding manifests and commit them to the GitOps repository.

This approach maintains ComponentRelease immutability, and the same release artifact deploys everywhere, with only configuration varying per environment.

### Rollback

To roll back a component, update the `releaseName` in the ReleaseBinding to point to the previous ComponentRelease name and push the change. OpenChoreo handles the rest.

---

## Clean Up

Remove the Argo CD Applications:

```bash
kubectl delete -f argo/root-application.yaml
```

This will delete the root Application and, because of the automated sync policy with pruning enabled, Argo CD will also clean up the child Applications and their managed resources.

> [!NOTE]
> If you want to remove Argo CD itself from the cluster, run:
> ```bash
> kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
> kubectl delete namespace argocd
> ```