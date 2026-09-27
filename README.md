# k8s-sre-platform

A Kubernetes-native SRE observability and reliability stack, deployed as plain
manifests via Kustomize. Runs on the cluster provisioned by
[`terraform-aws-eks-platform`](https://github.com/harsva/terraform-aws-eks-platform).

## Components

| Component     | Purpose                                                            |
| ------------- | ------------------------------------------------------------------ |
| Prometheus    | Metrics scraping, **SLO recording rules**, multi-burn-rate alerts  |
| Alertmanager  | Routing + **Slack** notification with severity-based inhibition    |
| Grafana       | Dashboards over Prometheus, HA (2 replicas) with HPA               |
| Workload policy | HPA, PodDisruptionBudgets, default-deny NetworkPolicies          |

## SRE features

- **SLO recording rules** compute error-ratio SLIs over 5m/30m/1h/6h windows.
- **Multi-window multi-burn-rate alerts** (Google SRE workbook): a fast-burn
  *page* (14.4x over 1h+5m) and a slow-burn *ticket* (6x over 6h+30m), so you
  alert on genuine error-budget burn rather than every transient spike.
- **Alertmanager inhibition** suppresses warnings when a critical alert for the
  same job is already firing — reducing alert noise.
- **HPA** scales Grafana on CPU + memory with tuned scale-up/scale-down behavior.
- **PodDisruptionBudgets** keep Prometheus (maxUnavailable: 0) and Grafana
  (minAvailable: 1) alive through node drains and rolling upgrades.
- **NetworkPolicies** default-deny ingress in `monitoring`, then allow intra-namespace.
- All workloads run **non-root**, drop all capabilities, use `RuntimeDefault`
  seccomp, and set resource requests/limits.

## Deploy

```bash
# Requires a cluster (see terraform-aws-eks-platform) and kubectl context set.
kubectl apply -k .

# Port-forward to explore locally:
kubectl -n monitoring port-forward svc/grafana 3000:3000
kubectl -n monitoring port-forward svc/prometheus 9090:9090
```

> Set a real Grafana admin password and Slack webhook before applying in a real
> environment. In production, source these from external-secrets backed by AWS
> Secrets Manager using the IRSA role from `terraform-aws-eks-platform`.

## Validation

CI validates every manifest against upstream Kubernetes schemas with
`kubeconform` (strict) and renders the Kustomize build. See
`.github/workflows/validate.yml`.

## Layout

```
namespaces/          monitoring namespace + ResourceQuota + LimitRange
prometheus/          RBAC, config, SLO/alert rules, StatefulSet
alertmanager/        config (Slack routing + inhibition) + Deployment
grafana/             datasource provisioning + Deployment
workload-policies/   HPA, PDBs, NetworkPolicies
kustomization.yaml   ties it all together
```
