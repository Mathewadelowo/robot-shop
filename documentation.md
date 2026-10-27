# Robot Shop on Amazon EKS — Capstone Project 2

A cloud-native microservices project that deploys the Robot Shop sample application on Amazon EKS and adds an observability workflow for metrics, dashboards, centralized logs, distributed traces, and Slack alert notifications.

## 1. Project Overview

Capstone Project 2 brings together a multi-service application and operational tooling to demonstrate how a containerized workload can be deployed, observed, and troubleshot on Kubernetes.

### Objectives

- Deploy and operate Robot Shop on Amazon EKS.
- Use Docker and Kubernetes to package and orchestrate application components.
- Manage application and observability configuration with Helm where configured.
- Collect and explore metrics with Prometheus and Grafana.
- Forward and query logs with Fluent Bit and Loki.
- Support distributed tracing with OpenTelemetry and Jaeger for instrumented requests.
- Route alerts through Alertmanager and deliver notifications to Slack.
- Document verification steps, troubleshooting, incident learnings, and cleanup.
- Define a 99.9% availability objective and calculate its theoretical error budget without confusing a target with measured performance.

### Key capabilities

- **Cloud platform:** AWS and Amazon EKS.
- **Containerization and orchestration:** Docker and Kubernetes.
- **Deployment configuration:** Helm, where the repository's deployment setup uses it.
- **Metrics and dashboards:** Prometheus and Grafana.
- **Logs:** Fluent Bit and Loki, subject to the active configuration.
- **Tracing:** OpenTelemetry and Jaeger for instrumented services.
- **Alerting:** Prometheus rules, Alertmanager routing, and Slack notifications.
- **Operational practice:** Runbook, postmortem notes, verification checklist, and secret-handling guidance.

## 2. Application Architecture
This diagram is conceptual. 

```text
                        Browser / User
                              |
                              v
                         Web front end
                              |
              +---------------+----------------+
              |               |                |
              v               v                v
          Catalogue          Cart              User
              |               |                |
              +---------------+----------------+
                              |
                     Application workflows
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          Payment          Shipping          Ratings
                              |
                              v
                           Dispatch

```



## 3. Observability Architecture

```text
Robot Shop workloads in Kubernetes
       |                 |                  |
       | metrics         | logs             | traces, when instrumented
       v                 v                  v
   Prometheus         Fluent Bit        OpenTelemetry
       |                 |                  |
       v                 v                  v
   Alert rules          Loki              Jaeger
       |
       v
   Alertmanager
       |
       v
   Slack #alerts

Grafana: dashboards and metrics exploration
```

This represents the intended architecture. Actual coverage depends on configured scrape targets, log inputs, instrumentation, collector configuration, and deployed resources.

### Prometheus and Grafana

Prometheus collects metrics from configured targets and evaluates `PrometheusRule` resources. Grafana is used to explore metrics and dashboards.

Verification should include checking that expected targets are discovered and healthy, alert rules are loaded, and dashboards show relevant data. A configured rule alone does not prove that every service is monitored.

### Alertmanager and Slack

During this project, the Slack notification path was tested:

- An `AlertmanagerConfig` named `robot-shop-slack` was configured in the `monitoring` namespace.
- The Slack receiver uses the Kubernetes Secret `slack-webhook`.
- The route matches `CatalogueServiceDown` and `SlackDeliveryTest`, with a namespace matcher for `monitoring`.
- The test rule was given the `namespace: monitoring` label to match the route.
- A Slack test notification was received successfully.

**What this demonstrates:** the controlled test alert reached Slack through the configured routing path. It does not prove that every service has a production alert, that a real outage occurred, or that the application meets an availability target.

Never commit the Slack webhook URL or print secret values in terminal output. If a webhook is exposed, revoke it and create a replacement.

### Catalogue availability alert example

The repository includes `monitoring/availability-alert.yaml`, which defines the `CatalogueServiceDown` alert. The rule checks the configured Prometheus `up` series for the Catalogue service in the `robot-shop` namespace and handles the absence of the expected series. It has a two-minute pending period and critical severity.

A possible PromQL query for the target is:

```promql
up{namespace="robot-shop", service="catalogue"}
```

Confirm the actual labels in Prometheus before relying on this exact query. A value of `1` means the target was scraped successfully at that evaluation time; `0` means the target was discovered but the scrape failed. An absent result may indicate that the target was not discovered or that the series is absent.

Catalogue is one service in Robot Shop. Extend alert coverage for other services based on project requirements and metrics that are actually available.

### Centralized logging: Fluent Bit and Loki

The repository includes `monitoring/fluent-bit-values.yaml` and `monitoring/loki-values.yaml`. Fluent Bit is intended to forward logs, while Loki provides storage and querying.

Verify which namespaces, pods, and containers are collected. Confirm the configured Loki endpoint and Fluent Bit output before changing or upgrading the deployment. The repository also contains a `fluentd` directory; do not assume Fluentd is part of the active Fluent Bit-to-Loki path without inspecting its manifests and purpose.

### Distributed tracing: OpenTelemetry and Jaeger

The repository contains tracing-related configuration and Jaeger evidence. OpenTelemetry and Jaeger can support collection and inspection of distributed traces for instrumented request paths.

Traces are only available when services are instrumented and spans are successfully exported to the configured collector/backend. Captured traces demonstrate specific examples, not complete tracing coverage across all services.

## 4. Technology Stack

| Technology | Purpose |
|---|---|
| AWS | Cloud platform |
| Amazon EKS | Managed Kubernetes |
| Terraform | Infrastructure as code, if the checked-out configuration manages the environment |
| Docker | Container image building |
| Kubernetes | Container orchestration |
| Helm | Application and observability configuration where configured |
| Prometheus | Metrics collection and alert evaluation |
| Grafana | Metrics visualization and exploration |
| Loki | Centralized log storage and querying |
| Fluent Bit | Log collection and forwarding |
| OpenTelemetry | Telemetry collection/export for instrumented services |
| Jaeger | Distributed trace inspection |
| Alertmanager | Alert grouping and notification routing |
| Slack | Operational notifications |
| Git and GitHub | Version control and collaboration |

## 5. Repository Guide

The following is a guide to known components and paths. Confirm exact paths in your current checkout before running commands.

```text
robot-shop/
├── web/
├── catalogue/
├── cart/
├── user/
├── payment/
├── shipping/
├── ratings/
├── dispatch/
├── mongo/
├── mysql/
├── load-gen/
├── fluentd/
├── monitoring/
│   ├── values.yaml
│   ├── loki-values.yaml
│   ├── fluent-bit-values.yaml
│   ├── availability-alert.yaml
│   ├── slack-test-alert.yaml
│   └── docs/
│       ├── runbook.md
│       └── postmortem.md
└── tracing/
```

The tree is representative, not exhaustive. A Terraform directory or Kubernetes Helm chart directory may exist in your checkout, but verify that before following any infrastructure or installation commands.

`monitoring/rendered.yaml`, if present locally, is generated Helm-rendered output for inspection. Do not apply it separately without confirming the intended deployment method, because it may conflict with resources managed by Helm. Inspect generated manifests for sensitive values before sharing or committing them.

The upstream `README.md` describes the sample application. Keep this capstone documentation separate unless you intentionally decide to edit the upstream README.

## 6. Environment Notes

The working environment described during this project included:

- **EKS cluster:** `voting-app-cluster`
- **AWS region:** `us-east-1`
- **Monitoring namespace:** `monitoring`
- **kube-prometheus-stack Helm release:** `monitoring`
- **Application namespace referenced by the alert:** `robot-shop`

These values describe the environment used during the work; they are not guarantees about your current AWS account or cluster. Verify your active AWS identity, Kubernetes context, namespaces, and Helm releases before making changes.

## 7. Getting Started and Verification

### Prerequisites

Install and configure the tools required by your environment:

- Git
- AWS CLI v2
- `kubectl`
- Helm 3
- Terraform, if provisioning infrastructure from this repository
- Docker, if building or changing application images
- An AWS account with the necessary permissions

Check tool versions:

```bash
git --version
aws --version
kubectl version --client
helm version
terraform version
docker --version
```

### Step 1: Open the repository

If you already have a checkout, use that directory. Otherwise, clone the repository URL supplied by your project owner or repository page:

```bash
git clone <repository-url>
cd <repository-directory>
```

Replace the placeholders with the actual repository URL and directory name.

### Step 2: Verify AWS access

Use your approved authentication method. For a local AWS CLI profile, you can run:

```bash
aws configure
aws sts get-caller-identity
aws configure get region
```

The working environment described in this project used `us-east-1`.

### Step 3: Provision infrastructure only if required

Only run Terraform if the Terraform configuration in your checkout manages the target infrastructure and the cluster does not already exist. First inspect the directory and its configuration. If it is present and correct, use:

```bash
terraform - <terraform-directory> init
terraform - <terraform-directory> validate
terraform - <terraform-directory> plan
```

Review the plan carefully before applying it. Replace `<terraform-directory>` with the actual path. Do not recreate an existing cluster just to repeat the setup.

### Step 4: Connect to EKS

For the working environment described above:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name voting-app-cluster
```

Verify the active context and cluster resources:

```bash
kubectl config current-context
kubectl get nodes
kubectl get namespaces
```

Use the correct cluster and AWS account for your environment.

### Step 5: Inspect Robot Shop

Inspect resources before installing or upgrading anything:

```bash
kubectl get deployments,pods,services -n robot-shop
helm list -A
```

For a fresh environment, inspect the actual chart path and values in the checkout before running `helm lint` or `helm install`. Do not assume a chart path exists without checking it, and do not install over an existing release.

### Step 6: Inspect the monitoring stack

The working environment used Helm release `monitoring` in namespace `monitoring`:

```bash
helm status monitoring -n monitoring
kubectl get pods,services -n monitoring
kubectl get prometheusrule,servicemonitor -n monitoring
```

Review the current Helm release values and chart version before upgrading. Helm changes can affect live resources.

### Step 7: Verify services and alert rules

```bash
kubectl get deployments,pods,services -n robot-shop
kubectl get prometheusrule -n monitoring
```

Check the Catalogue service and endpoints if those names match the deployed resources:

```bash
kubectl get svc catalogue -n robot-shop
kubectl get endpoints catalogue -n robot-shop
```

In Prometheus, inspect the `up` series and confirm the actual labels before using a query such as:

```promql
up{namespace="robot-shop", service="catalogue"}
```

Use the same verification approach for other services, adapting queries to metrics and targets that are actually configured.

### Step 8: Verify logging

Inspect the relevant resources:

```bash
kubectl get pods,services,daemonsets -n monitoring
```

Confirm the Fluent Bit output and Loki endpoint in the deployed values. In Grafana, select the configured Loki data source and use **Explore** to query collected logs.

### Step 9: Verify tracing

Inspect the repository files and deployed resources:

```bash
find tracing -maxdepth 2 -type f
kubectl get pods,services -n monitoring
```

Confirm the actual Jaeger service name and UI port before starting a port-forward. Jaeger can display traces only when requests are instrumented and spans are successfully exported.

### Step 10: Verify Slack alerting

Check resource existence without printing secret contents:

```bash
kubectl get secret slack-webhook -n monitoring
kubectl get alertmanagerconfig -n monitoring
kubectl get prometheusrule -n monitoring
```

The project work confirmed that a controlled `SlackDeliveryTest` notification was received. The `CatalogueServiceDown` rule also needed the namespace label required by the configured Alertmanager route.

A successful test notification validates the test delivery path. It does not establish that every service has an alert or that Catalogue is currently down.

## 8. Useful Port-Forward Commands

First check actual service names and ports:

```bash
kubectl get services -n monitoring
```

Run each port-forward in a separate terminal and leave it running while using the local interface. The following service names are examples; use the names returned by your cluster.

### Prometheus

```bash
kubectl port-forward -n monitoring \
  svc/<prometheus-service-name> 9090:9090
```

Open `http://localhost:9090`.

### Alertmanager

```bash
kubectl port-forward -n monitoring \
  svc/<alertmanager-service-name> 9093:9093
```

Open `http://localhost:9093`.

### Grafana

Use the actual Grafana service name and service port. For example, if the service is `monitoring-grafana` and exposes port `80`:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-grafana 3000:80
```

Open `http://localhost:3000` and use the credentials configured for your deployment. Do not put passwords in this document.

### General health checks

```bash
kubectl get pods -n robot-shop
kubectl get services -n robot-shop
kubectl get pods -n monitoring
```

If the workload is deployed as `deployment/catalogue`, its logs can be inspected with:

```bash
kubectl logs -n robot-shop deployment/catalogue
```

Confirm the actual deployment name first.

## 9. Availability SLO and Error Budget

The project documents a **99.9% availability objective over a rolling 30-day window**.

| Measurement | Value |
|---|---:|
| Availability objective | 99.9% |
| Allowed unavailability | 0.1% |
| Minutes in 30 days | 43,200 |
| Theoretical error budget at 99.9% | 43.2 minutes |

Calculation:

```text
30 days × 24 hours × 60 minutes = 43,200 minutes
43,200 minutes × 0.001 = 43.2 minutes
```

This is the theoretical error budget for the stated window and target. It is not a measured result.

**Current limitation:** the Catalogue alert detects scrape failure or an absent target series, but a validated rolling 30-day SLO calculation and error-budget consumption metric remain to be implemented. Define the service boundary and availability signal before reporting SLO compliance or extending the objective to the entire application.


## 10. Troubleshooting Guide

| Symptom | Checks |
|---|---|
| Pod is not ready | Run `kubectl describe pod <pod-name> -n <namespace>` and inspect events, probes, image pulls, and resource limits. |
| Service has no endpoints | Check pod labels/selectors and readiness with `kubectl get pods --show-labels -n <namespace>` and `kubectl describe svc <service-name> -n <namespace>`. |
| Prometheus target is down or missing | Check target discovery, ServiceMonitor/PodMonitor selectors, namespace labels, service ports, and scrape configuration. |
| Alert is firing but Slack receives nothing | Check the alert labels, Alertmanager route matchers, receiver configuration, Secret existence, and Alertmanager logs. Never print the webhook value. |
| No logs appear in Loki | Check Fluent Bit pod status and output configuration, Loki readiness, label selectors, and Grafana's selected Loki data source. |
| No traces appear in Jaeger | Check service instrumentation, exporter endpoint, collector health, sampling, and whether spans are being generated. |
| Helm upgrade behaves unexpectedly | Inspect `helm status`, `helm history`, current values, and the proposed diff/plan before making another change. |

Use `monitoring/docs/runbook.md` if it exists in your checkout for environment-specific troubleshooting steps. Record significant incidents, root causes, and follow-up actions in `monitoring/docs/postmortem.md` if that file is maintained by the project.

## 11. Security and Cost Management

- **Credentials:** Use approved AWS authentication and Kubernetes Secrets or an approved secret manager for sensitive values.
- **Git hygiene:** Do not commit webhook URLs, tokens, passwords, private keys, local credentials, Terraform state, or generated manifests containing secrets.
- **Secret exposure:** If a webhook or credential is exposed, revoke/rotate it and review repository history and logs for further exposure.
- **Infrastructure cost:** EKS, worker nodes, storage, load balancers, and telemetry retention may incur ongoing charges.
- **Resource usage:** Set appropriate resource requests/limits and configure storage and retention intentionally.
- **Production readiness:** Review access control, network exposure, encryption, backups, and alert coverage before production use.
- **Sample application limitations:** The upstream Robot Shop README describes the application as a learning sandbox, not a production-secure reference without additional hardening.

## 12. Cleanup

> **Warning:** Cleanup can permanently remove workloads, data, storage, and cloud infrastructure. Confirm the AWS account, Kubernetes context, namespace, and resource ownership first.

Inspect releases and persistent volume claims:

```bash
helm list -A
kubectl get pvc -A
```

Only uninstall releases that you have confirmed belong to this project and are no longer needed. For example, if those exact releases exist and are safe to remove:

```bash
helm uninstall robot-shop -n robot-shop
helm uninstall monitoring -n monitoring
```

These are examples, not commands to run blindly. Loki may be a separate Helm release, and removing a release does not necessarily remove persistent volumes or every cloud resource.

If Terraform manages the infrastructure, inspect the destroy plan from the correct directory:

```bash
terraform -chdir=<terraform-directory> plan -destroy
```

Run `terraform destroy` only when you intentionally want to delete the Terraform-managed environment and have verified the plan. Check for AWS resources created outside Terraform separately.

## 13. Project Documentation and Evidence

- `monitoring/docs/runbook.md` — operational troubleshooting, if present.
- `monitoring/docs/postmortem.md` — incident analysis and follow-up actions, if present.
- `monitoring/availability-alert.yaml` — Catalogue availability alert example.
- `monitoring/slack-test-alert.yaml` — Slack delivery test rule.
- `tracing/` — tracing configuration and captured evidence.
- `monitoring/` values files — monitoring, Loki, and Fluent Bit configuration.

Ensure screenshots used in a report do not expose webhook URLs, passwords, tokens, account identifiers, or other secrets.

## 14. Project Status Summary

Capstone Project 2 combines the Robot Shop microservices workload with EKS/Kubernetes operations and an observability stack. The Slack test notification was verified, and the Catalogue availability rule is documented as one example of service-level alerting.

Prometheus target coverage, Grafana dashboards, Loki log ingestion, Jaeger trace coverage, and service-specific alerts should be reported as complete only to the extent they have been independently verified in the target environment. The rolling SLO measurement remains unfinished and must not be presented as measured availability.
"""


