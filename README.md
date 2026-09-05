# Product API GitOps Platform

This repository is the declarative delivery layer for the Product API. Argo CD continuously reconciles the desired state held in Git to Kubernetes.

## Platform architecture

```
Application repository -> GitHub Actions -> GHCR image
                                      |
                                      v
GitOps repository -> Argo CD -> Kubernetes environments
```

## Environments

| Environment | Namespace | Sync policy | Purpose |
| --- | --- | --- | --- |
| Dev | `product-api-dev` | Automatic + self-healing | Continuous integration |
| Staging | `product-api-staging` | Manual | Release validation |
| Production | `product-api-production` | Manual | Approved releases |

## Bootstrap a local cluster

Prerequisites: Docker, kubectl, Helm, an active Kubernetes cluster, and Argo CD.

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \\
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait --for=condition=Available deployment/argocd-server --timeout=300s

kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application-dev.yaml
```

Check reconciliation:

```bash
kubectl get applications -n argocd
```

## Image promotion and rollback

The `image-published` repository-dispatch event updates an overlay with an immutable image digest. Configure a `GITOPS_REPO_TOKEN` secret in the application repository before enabling cross-repository promotion.

To demonstrate rollback, revert the image-promotion commit in this repository. Argo CD automatically reconciles the previous desired state for development.

## Production note

Before adding any third-party platform component, verify its exact version and licence in Oracle-approved third-party tracking.
