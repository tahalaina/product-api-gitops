# Product API GitOps Platform

GitOps delivery repository for the Product API. Argo CD watches this repository and continuously reconciles Kubernetes to the desired state committed in Git.

## Architecture

```
Application repository -> GitHub Actions -> GHCR image
                                     |
                                     v
                          repository_dispatch event
                                     |
                                     v
GitOps repository -> image tag commit -> Argo CD -> Kubernetes (dev/staging/production)
```

## Environments

| Environment | Namespace | Sync policy | Purpose |
| --- | --- | --- | --- |
| Dev | `product-api-dev` | Automatic | Continuous integration |
| Staging | `product-api-staging` | Manual | Release validation |
| Production | `product-api-production` | Manual | Approved releases |

## Local end-to-end demo

Prerequisites: Docker, kubectl, Helm, and a Kubernetes cluster (Kind, Minikube, or Docker Desktop Kubernetes).

1. Create a cluster and point `kubectl` to it.
2. Install Argo CD:

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side --force-conflicts \\
     -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl -n argocd wait --for=condition=Available deployment/argocd-server --timeout=300s
   ```

3. Apply the project and development application:

   ```bash
   kubectl apply -f argocd/project.yaml
   kubectl apply -f argocd/application-dev.yaml
   ```
4. Check reconciliation:

   ```bash
   kubectl get applications -n argocd
   kubectl get pods -n product-api-dev
   ```

## Image promotion

The `image-published` repository-dispatch event updates the relevant overlay with an immutable image digest. Configure a `GITOPS_REPO_TOKEN` secret in the application repository and trigger the event after an image is published.

To demonstrate rollback, revert the promotion commit in this repository. Argo CD reconciles the prior desired state automatically for `dev`.

Never commit real credentials. `apps/product-api/base/database-secret.yaml` contains demo-only local credentials; replace this with External Secrets, Sealed Secrets, or your cloud secret manager before any real deployment.
