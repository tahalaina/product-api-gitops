# Product API GitOps and Observability Platform

This repository is the declarative delivery and observability layer for the Product API. Argo CD continuously reconciles the desired state held in Git to Kubernetes.

## Platform architecture

```
Application repository -> GitHub Actions -> GHCR image
                                      |
                                      v
GitOps repository -> Argo CD -> Kubernetes environments
                                  |
             +--------------------+--------------------+
             |                    |                    |
         Prometheus            Loki + OTel          Tempo
         metrics + alerts      centralized logs      distributed traces
             \                    |                    /
              +---------------- Grafana -------------+
                         dashboard and exploration
```

## Environments

| Environment | Namespace | Sync policy | Purpose |
| --- | --- | --- | --- |
| Dev | `product-api-dev` | Automatic + self-healing | Continuous integration |
| Staging | `product-api-staging` | Manual | Release validation |
| Production | `product-api-production` | Manual | Approved releases |

## Observability

| Component | Role |
| --- | --- |
| Prometheus + Alertmanager | Scrapes Spring Boot `/actuator/prometheus` metrics and evaluates availability/5xx alerts. |
| Grafana | Supplies the Product API dashboard with availability, P95 latency, error rate, CPU and memory panels. |
| Loki | Stores centralized Kubernetes container logs. |
| OpenTelemetry Collector | Receives OTLP traces from the API and forwards pod logs to Loki. |
| Tempo | Stores traces sent by the collector and makes them searchable in Grafana. |

The Product API has Micrometer Prometheus metrics and Micrometer/OpenTelemetry tracing enabled. The dashboard and alerts are committed as Kubernetes resources, not created manually in the UI.

## Bootstrap a local cluster

Prerequisites: Docker, kubectl, Helm, an active Kubernetes cluster, and Argo CD.

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \\
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait --for=condition=Available deployment/argocd-server --timeout=300s

kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application-dev.yaml
kubectl apply -f argocd/application-observability-metrics.yaml
kubectl apply -f argocd/application-observability-loki.yaml
kubectl apply -f argocd/application-observability-tempo.yaml
kubectl apply -f argocd/application-observability-otel.yaml
kubectl apply -f argocd/application-observability-resources.yaml
```

Check reconciliation:

```bash
kubectl get applications -n argocd
kubectl get pods -n observability
```

For local access, port-forward Grafana and retrieve its generated admin password:

```bash
kubectl -n observability port-forward svc/monitoring-grafana 3000:80
kubectl -n observability get secret monitoring-grafana \\
  -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

Open `http://localhost:3000`, then choose **Dashboards → Product API → Product API — Service Overview**. Grafana also includes Loki and Tempo data sources for log and trace exploration.

## Image promotion and rollback

The `image-published` repository-dispatch event updates an overlay with an immutable image digest. Configure a `GITOPS_REPO_TOKEN` secret in the application repository before enabling cross-repository promotion.

To demonstrate rollback, revert the image-promotion commit in this repository. Argo CD automatically reconciles the previous desired state for development.

## Production note

The included Loki and Tempo configurations are intentionally lightweight for a local portfolio cluster. A production deployment should use managed object storage, secret management, retention policies, backups, TLS, authentication, and alert-notification routing.
