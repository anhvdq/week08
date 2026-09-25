# Helm monitoring setup and evidence

Helm manages the shared `monitoring` release on the existing AKS cluster.
Prometheus scrapes kubelet/cAdvisor, node-exporter, and kube-state-metrics.
No application code or metrics endpoint is required. Grafana includes provisioned
Kubernetes dashboards. Open **Kubernetes / Compute Resources / Namespace (Pods)**
and select staging or production for per-pod CPU and memory. Other built-in
workload dashboards show readiness and restarts.

## Setup

Set staging environment secret `GRAFANA_ADMIN_PASSWORD` to a strong password.
The existing `AZURE_CREDENTIALS` identity needs AKS access sufficient to install
cluster-scoped monitoring resources and CRDs. Monitoring uses Helm directly;
Azure infrastructure provisioning uses the separate Terraform workflow. Docker Scout still requires repository secrets
`DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.

Check capacity and `kubectl get storageclass managed-csi` before deploying.
Prometheus requests 512 MiB memory and a 10 GiB disk; allow additional room for
Grafana, the operator and exporters. Dashboards are provisioned from the chart;
Grafana UI edits are ephemeral. Prometheus retains 24 hours of system metrics.

CI renders the pinned chart before image publishing. The staging workflow calls
`.github/workflows/deploy-monitoring.yml`, which creates the monitoring namespace
and admin secret, then runs `helm upgrade --install`. Staging application deployment
waits for that reusable workflow to succeed. Both use the commit that passed CI. `--atomic --wait` fails the pipeline and rolls back
a failed upgrade. Staging concurrency serializes release updates. The shared stack
monitors both application namespaces. Production verification is dispatched by
the existing production deployment workflow.

The chart is pinned to 75.15.1 and Helm to 3.18.4. When changing chart versions,
review the chart's CRD upgrade instructions before deploying. Do not install over
an unrelated existing release; inspect `helm list -n monitoring` first.

## Access and validation

```bash
kubectl -n monitoring get pods,pvc
helm status monitoring -n monitoring
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
# Separate terminal, if needed:
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open Grafana at http://localhost:3000 and sign in as admin with the configured
password. Both services use ClusterIP; no public monitoring endpoint is created.
Capture populated namespace dashboard screenshots for both environments.
At http://localhost:9090 inspect Targets and query
`container_cpu_usage_seconds_total{namespace="staging"}`.
Use Prometheus and Grafana to manually confirm CPU, memory, readiness and
restart metrics for the application containers. Deployment rollout checks
establish workload readiness.

Retain pipeline links/logs, Helm status, Scout reports, successful staging and
production tests, Prometheus queries, and dashboard screenshots. Live deployment
and vulnerability remediation evidence must come from actual runs.
For a failed release inspect `helm history monitoring -n monitoring` and pod logs;
use `helm rollback monitoring <revision> -n monitoring --wait` if appropriate.
Do not delete monitoring PVCs to repair a failed release.

## Infrastructure

See [infrastructure.md](infrastructure.md) for the Terraform workflow and remote
state setup. Infrastructure provisioning completes before image publishing and
monitoring deployment.
