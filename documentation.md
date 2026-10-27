# Robot Shop on Amazon EKS --- Capstone Project Documentation

## 1. Project Overview

This project deploys the Robot Shop microservices application on Amazon
Elastic Kubernetes Service (Amazon EKS) and documents metrics,
dashboards, logging, tracing, alerting, and operations.

### Objectives

-   Deploy and operate the application on Kubernetes.
-   Monitor traffic, errors, latency, pod restarts, CPU, and memory.
-   Collect and query application logs.
-   Enable distributed tracing where instrumentation is configured.
-   Configure alerting and validate notification delivery.
-   Document deployment verification, troubleshooting, security, and
    cleanup.

### Technology Stack

  -----------------------------------------------------------------------
  Area                                Technology
  ----------------------------------- -----------------------------------
  Cloud                               AWS

  Kubernetes                          Amazon EKS

  Containers                          Docker

  Deployment                          Kubernetes, Helm

  Metrics and dashboards              Prometheus, Grafana

  Logs                                Fluent Bit, Loki

  Distributed tracing                 OpenTelemetry, Jaeger

  Alerting                            Prometheus Alertmanager,
                                      AlertmanagerConfig, Slack webhook

  Infrastructure as code              Terraform
  -----------------------------------------------------------------------

Verify the current manifests and deployed resources before treating
every listed component as active in the environment.

## 2. Architecture

### Application architecture

``` text
Users
  |
  v
Kubernetes entry point / ingress (if configured)
  |
  v
Robot Shop frontend
  |
  +--> Catalogue
  +--> Cart
  +--> User
  +--> Payment
  +--> Shipping
  +--> Ratings
  +--> Supporting services
```

The exact services and traffic paths depend on the manifests deployed in
the cluster.

### Observability architecture

``` text
Application and Kubernetes workloads
  |
  +--> Prometheus --> Grafana dashboards
  |         |
  |         +--> Alertmanager --> Slack notifications (when configured)
  |
  +--> Fluent Bit --> Loki --> Log queries
  |
  +--> OpenTelemetry instrumentation/collector --> Jaeger (when configured)
```

## 3. Repository Guide

Review the application service directories and Kubernetes configuration,
Helm chart and values files, Prometheus/Grafana configuration, Fluent
Bit/Loki settings, alert rules, tracing configuration, and Terraform
files where present.

Monitoring-related files referenced during project work include: -
`values.yaml` - `loki-values.yaml` - `fluent-bit-values.yaml` -
`availability-alert.yaml` - `slack-test-alert.yaml` -
`docs/runbook.md` - `docs/postmortem.md` - `tracing/`

Check the repository for the exact paths and filenames in the current
branch.

## 4. Environment and Prerequisites

  Setting                                  Value
  ---------------------------------------- ----------------------
  AWS region                               `us-east-1`
  EKS cluster name used in the project     `voting-app-cluster`
  Application namespace                    `robot-shop`
  Monitoring namespace                     `monitoring`
  Helm release referenced in the project   `monitoring`

These are the documented project values. Verify them against your AWS
account before running commands.

### Prerequisites

Install and configure AWS CLI, `kubectl`, Helm, Git, and appropriate IAM
permissions for EKS access.

``` bash
aws eks update-kubeconfig --region us-east-1 --name voting-app-cluster
kubectl get nodes
```

Replace the cluster name or region if your environment differs.

## 5. Deployment Verification

``` bash
kubectl get nodes
kubectl get namespaces
kubectl get pods -n robot-shop
kubectl get pods -n monitoring
kubectl get deployments -n robot-shop
kubectl get services -n robot-shop
kubectl get deployments -n monitoring
kubectl get services -n monitoring
helm list -A
```

Inspect the monitoring Helm release:

``` bash
helm status monitoring -n monitoring
```

Check custom monitoring resources:

``` bash
kubectl get servicemonitors -A
kubectl get prometheusrules -A
kubectl get alertmanagerconfigs -A
```

A resource may be absent if the relevant chart, custom resource
definition, or configuration has not been installed. Treat command
output as the source of truth for the current cluster.

## 6. Prometheus and Grafana

The examples below use catalogue metric names seen during project
validation. If the metrics or labels differ, inspect the metrics
available in Prometheus and adapt the queries.

### Requests per second

``` promql
sum(rate(catalogue_http_requests_total{route!="/metrics"}[5m]))
```

Calculates the per-second request rate over five minutes, excluding the
metrics endpoint.

### HTTP 5xx errors per second

``` promql
sum(rate(catalogue_http_requests_total{route!="/metrics",status=~"5.."}[5m])) or vector(0)
```

Calculates the rate of HTTP 5xx responses. `or vector(0)` can display
zero when no matching series exists; verify the metric and labels before
interpreting that as proof of zero errors.

### 95th-percentile request latency

``` promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(catalogue_http_request_duration_seconds_bucket{route!="/metrics"}[5m])
  )
)
```

Estimates p95 request duration from histogram buckets. The result is in
seconds if the histogram uses seconds.

### Container restarts over the last 15 minutes

``` promql
sum by (pod) (
  increase(kube_pod_container_status_restarts_total{
    namespace="robot-shop"
  }[15m])
)
```

If filtering with `> 0` returns no data, it may mean no restarts
occurred in the window. First check that the underlying metric exists:

``` promql
kube_pod_container_status_restarts_total{namespace="robot-shop"}
```

### CPU usage by pod

``` promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="robot-shop",
    container!="",
    image!=""
  }[5m])
)
```

Estimates CPU cores used by containers, grouped by pod. Metric
availability depends on the cluster's metrics pipeline.

### Memory usage by pod

``` promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="robot-shop",
    container!="",
    image!=""
  }
)
```

Returns working-set memory in bytes, grouped by pod.

### Request counts versus time series

``` promql
count by (route) (catalogue_http_request_duration_seconds_bucket)
```

This counts matching time series per route; it does **not** count HTTP
requests. To estimate request volume over a period, use a counter
increase query:

``` promql
sum(increase(catalogue_http_requests_total{route!="/metrics"}[15m]))
```

During earlier validation, this returned approximately 10 requests over
15 minutes, while the 5xx rate query returned zero matching errors for
that window. These are point-in-time observations, not guarantees about
future traffic or overall reliability.

### Access Grafana

If the service is named `monitoring-grafana` and exposes port 80:

``` bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Then open `http://localhost:3000`. Confirm the service name and port
with `kubectl get svc -n monitoring` if needed. Use configured
credentials and do not commit them to Git.

## 7. Logging with Fluent Bit and Loki

Fluent Bit can collect container logs and forward them to Loki. Check
logging components:

``` bash
kubectl get pods -n monitoring
kubectl get services -n monitoring
```

Inspect Fluent Bit logs if records are missing:

``` bash
kubectl logs -n monitoring -l app.kubernetes.io/name=fluent-bit --tail=100
```

The label selector may differ by chart version. Use
`kubectl get pods -n monitoring --show-labels` to identify the correct
selector.

The Loki gateway address referenced in this project is:

``` text
loki-gateway.monitoring.svc.cluster.local:80
```

Verify the service exists before relying on that address. Review Loki
persistence and retention settings before using the stack for long-term
logging. If persistence is disabled, logs may not survive pod
replacement or storage lifecycle events.

## 8. Distributed Tracing with OpenTelemetry and Jaeger

Distributed tracing helps follow requests across services and identify
latency or errors. OpenTelemetry and Jaeger are included in the
project's observability design/configuration.

Verify tracing resources:

``` bash
kubectl get pods -n monitoring
kubectl get services -n monitoring
kubectl get pods -A | grep -Ei 'otel|opentelemetry|jaeger'
```

Tracing requires application instrumentation or trace generation, a
configured exporter/collector path, and a reachable trace backend.
Installing a backend alone does not prove that application traces are
being generated or received.

## 9. Alerting and Slack Notifications

Project configuration references include `availability-alert.yaml`,
`slack-test-alert.yaml`, an `AlertmanagerConfig` named
`robot-shop-slack`, and a Kubernetes Secret named `slack-webhook`.

Check current resources:

``` bash
kubectl get prometheusrules -A
kubectl get alertmanagerconfigs -A
kubectl get secrets -n monitoring
```

Do not print, paste, or commit the Slack webhook URL or any other secret
value.

### Alert validation

1.  Confirm the Prometheus rule is installed and loaded.
2.  Confirm Alertmanager is running and discovers the notification
    configuration.
3.  Confirm the Slack webhook Secret exists and is referenced correctly.
4.  Trigger only a controlled test alert.
5.  Verify delivery in the intended Slack channel.
6.  Remove or disable test-only alerts after validation.

A controlled Slack test alert was reported as successful during project
work. This confirms that the tested path worked at that time, not that
every future alert will be delivered.

## 10. Availability SLO and Error Budget

A 99.9% availability target over 30 days allows a theoretical
unavailability budget of approximately **43.2 minutes**: - Total time in
30 days: 43,200 minutes. - 0.1% of 43,200 minutes: 43.2 minutes.

I implemented Service Level Indicator (SLI) and Service Level Objective (SLO) monitoring for the Robot Shop catalogue service using Prometheus and Grafana. The SLI measures the percentage of successful HTTP requests, while the SLO defines a target of 99.9% over a rolling 30-day window. A PromQL comparison evaluates whether the service meets the target, and Grafana visualizes the result

## 11. Troubleshooting

  ------------------------------------------------------------------------------------------------
  Symptom                 Checks                                           Possible next step
  ----------------------- ------------------------------------------------ -----------------------
  Pods are pending        `kubectl describe pod <pod> -n <namespace>`      Inspect scheduling
                                                                           events, resource
                                                                           requests, and node
                                                                           capacity.

  Pods are restarting     `kubectl logs <pod> -n <namespace> --previous`   Inspect previous
                                                                           container logs and pod
                                                                           events.

  Grafana has no data     Check Prometheus targets and the selected data   Verify scrape
                          source                                           configuration, labels,
                                                                           and time range.

  A PromQL query returns  Query the raw metric name first                  Confirm metric/label
  no series                                                                names, namespace, and
                                                                           scrape status.

  Loki has no logs        Inspect Fluent Bit logs and Loki services        Check output
                                                                           configuration,
                                                                           connectivity, labels,
                                                                           and storage.

  Traces are missing      Inspect instrumentation and collector/exporter   Verify endpoint,
                          logs                                             protocol, sampling, and
                                                                           backend connectivity.

  Slack alerts are not    Inspect Alertmanager logs and configuration      Verify routing,
  delivered                                                                receiver settings,
                                                                           Secret reference, and
                                                                           webhook validity
                                                                           without exposing the
                                                                           secret.

  A resource is not found `kubectl get <resource> -A`                      Confirm namespace,
                                                                           installation status,
                                                                           and CRD availability.
  ------------------------------------------------------------------------------------------------

Consult `monitoring/docs/runbook.md` in the current repository.

## 12. Security and Cost Considerations

### Security

-   Never commit AWS credentials, kubeconfig files, Slack webhook URLs,
    passwords, or other secrets.
-   Use IAM roles and least-privilege access where possible.
-   Restrict access to Grafana, Prometheus, Loki, Jaeger, and Kubernetes
    APIs.
-   Avoid publicly exposing internal observability services without
    suitable authentication and access controls.
-   Rotate any secret that may have been exposed.
-   Review image sources, versions, and permissions before production
    use.

### Cost

-   Monitor EKS, EC2, load balancer, storage, logging, and data-transfer
    costs.
-   Set appropriate CPU and memory requests/limits.
-   Configure log retention and sampling to manage storage and ingestion
    costs.
-   Remove unused resources after testing.
-   Confirm whether persistence is enabled and how much storage is
    provisioned.

## 13. Evidence and Project Limitations

Capture evidence from the actual environment for a complete capstone
submission. Useful evidence includes: - Running application pods and
services. - Helm release status. - Prometheus targets showing expected
scrape targets as healthy. - Grafana panels for traffic, errors,
latency, restarts, CPU, and memory. - Loki queries returning relevant
application logs. - Jaeger showing traces from instrumented requests, if
tracing is active. - Alert rules and a successful controlled Slack
notification. - Terraform plan/apply output, if Terraform was used to
provision infrastructure.

Do not use a single screenshot or test as proof of continuous
availability. Record the date, namespace, relevant configuration, and
test conditions for each item of evidence.

## 14. Cleanup

Only run cleanup commands when you are certain the resources are no
longer needed. These commands can remove live workloads or
infrastructure.

If monitoring was installed as the Helm release shown here:

``` bash
helm uninstall monitoring -n monitoring
```

If appropriate, remove the application namespace:

``` bash
kubectl delete namespace robot-shop
```

Delete AWS infrastructure only through the actual infrastructure-as-code
workflow after reviewing dependencies and data-retention requirements.
Do not delete a shared or production cluster just to clean up this
capstone environment.

## 15. Conclusion

This capstone brings together Kubernetes deployment and observability
practices for Robot Shop on Amazon EKS. Prometheus and Grafana support
metrics and dashboards, Fluent Bit and Loki support log collection and
querying, OpenTelemetry and Jaeger support distributed tracing when
properly instrumented, and Alertmanager with Slack supports notification
workflows.

The final submission should distinguish implemented and tested features
from planned or partially configured components, include evidence from
the actual environment, and document remaining gaps.



##  Project Documentation and Evidence

- `monitoring/docs/runbook.md` — operational troubleshooting.
- `monitoring/docs/postmortem.md` — incident analysis and follow-up actions.
- `monitoring/availability-alert.yaml` — Catalogue availability alert example.
- `monitoring/slack-test-alert.yaml` — Slack delivery test rule.
- `tracing/` — tracing configuration and captured evidence.
- `monitoring/` values files — monitoring, Loki, and Fluent Bit configuration.
.

"""


