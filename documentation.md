# 🚀 Robot Shop on Amazon EKS

A cloud-native microservices project that deploys Robot Shop to Amazon EKS and integrates metrics monitoring, centralized logging, distributed tracing, and Slack alerting.

---

## 📌 Overview

Robot Shop demonstrates how to deploy and observe a containerized microservices application using AWS and Kubernetes.

The project combines infrastructure provisioning, Helm-based application deployment, metrics collection, log aggregation, distributed tracing, and alert notifications in a single DevOps workflow.

It also documents a **99.9% availability Service Level Objective (SLO)** and an error-budget policy for the Catalogue service.

### 🌟 Key Features

- **☁️ Cloud infrastructure:** Terraform and Amazon EKS.
- **🐳 Containerization:** Docker-based application images.
- **☸️ Orchestration:** Kubernetes workloads, services, namespaces, and health checks.
- **📦 Application deployment:** Helm-based Robot Shop deployment.
- **📊 Metrics monitoring:** Prometheus metrics collection and alert evaluation.
- **📈 Visualization:** Grafana dashboards and observability exploration.
- **📝 Centralized logging:** Loki log storage and Fluent Bit log forwarding.
- **🔍 Distributed tracing:** Jaeger and OpenTelemetry Collector integration.
- **🚨 Alert management:** Prometheus alert rules and Alertmanager routing.
- **💬 Slack notifications:** Incoming webhook integration with the `#alerts` channel.
- **🎯 Availability policy:** A documented Catalogue availability target and error-budget policy.
- **🔐 Configuration safety:** Kubernetes Secrets for sensitive notification credentials.

> **Implementation status:** The Catalogue alert rule and Slack notification path have been configured and tested. A rolling 30-day SLO calculation remains a planned improvement; do not report measured SLO compliance until it is implemented and validated.

---

## 🏗️ Architecture

```text
Developer
   |
   v
GitHub repository
   |
   v
Amazon EKS
   |
   +----------------------------+
   |                            |
   v                            v
robot-shop namespace       monitoring namespace
   |                            |
   v                            +---- Prometheus
Robot Shop services             |        |
   |                            |        v
   v                            |    Alertmanager ---> Slack #alerts
Catalogue service               |
                                +---- Grafana
                                |
                                +---- Loki <--- Fluent Bit
                                |
                                +---- OpenTelemetry Collector
                                             |
                                             v
                                           Jaeger
```

This diagram shows the intended observability components. Actual metric, log, and trace coverage depends on the deployed configuration and application instrumentation.

---

## 🛠️ Technology Stack

| Technology | Purpose |
|---|---|
| AWS | Cloud infrastructure |
| Amazon EKS | Managed Kubernetes |
| Terraform | Infrastructure as code |
| Docker | Container image building |
| Kubernetes | Container orchestration |
| Helm | Application and observability deployments |
| Prometheus | Metrics collection and alert evaluation |
| Grafana | Metrics and log visualization |
| Loki | Centralized log storage |
| Fluent Bit | Log collection and forwarding |
| OpenTelemetry | Telemetry collection and export |
| Jaeger | Distributed trace visualization |
| Alertmanager | Alert grouping and notification routing |
| Slack | Operational alert notifications |
| Git and GitHub | Source control and collaboration |

---

## 📂 Repository Structure

The structure below is representative. Confirm paths against the files in your checkout before running the commands.

```text
robot-shop/
├── catalogue/
├── K8s/
│   └── helm/
├── monitoring/
│   ├── values.yaml
│   ├── loki-values.yaml
│   ├── fluent-bit-values.yaml
│   ├── availability-alert.yaml
│   ├── slack-test-alert.yaml
│   ├── rendered.yaml
│   └── docs/
│       ├── runbook.md
│       └── postmortem.md
├── tracing/
├── terraform-eks/
├── fluentd/
├── README.md
└── README (2).md
```

The original `README.md` belongs to the upstream project. This document is a separate capstone/project README.

---

## 🚀 Getting Started

### Prerequisites

Install and configure:

- Git
- AWS CLI v2
- `kubectl`
- Helm 3
- Terraform (if provisioning infrastructure)
- Docker (if building or changing application images)
- An AWS account with permissions for the required infrastructure

Check the installed tools:

```bash
git --version
aws --version
kubectl version --client
helm version
terraform version
docker --version
```

### 1. Clone the repository

```bash
git clone https://github.com/Mathewadelowo/robot-shop.git
cd robot-shop
```

If you already have a local checkout, use that directory instead of cloning again.

### 2. Configure AWS access

Use your organization's approved authentication method. For a local profile, you can run:

```bash
aws configure
aws sts get-caller-identity
aws configure get region
```

The project environment described during this work uses region `us-east-1`.

### 3. Provision infrastructure, if needed

Only do this if the Terraform configuration in your checkout manages the target infrastructure and the cluster does not already exist.

```bash
cd terraform-eks
terraform init
terraform validate
terraform plan
```

Review the plan carefully before applying it:

```bash
terraform apply
```

Then return to the repository root:

```bash
cd ..
```

If the EKS cluster already exists, do not recreate it; proceed to the next step.

### 4. Connect `kubectl` to EKS

The cluster used during this project is `voting-app-cluster` in `us-east-1`:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name voting-app-cluster
```

Verify the connection:

```bash
kubectl config current-context
kubectl get nodes
kubectl get namespaces
```

### 5. Check the Robot Shop deployment

Before installing anything, inspect the current release and resources:

```bash
helm list -n robot-shop
kubectl get deployments,pods,services -n robot-shop
```

If you are deploying into a new environment, first validate the chart path and its values:

```bash
helm lint ./K8s/helm
```

Only install the chart after confirming the chart path, required values, namespace, and Redis/storage configuration for your version. Avoid running `helm install` over an existing release.

### 6. Check the monitoring stack

The monitoring Helm release used in this project is named `monitoring` in namespace `monitoring`.

```bash
helm status monitoring -n monitoring
kubectl get pods,services -n monitoring
kubectl get prometheusrule,servicemonitor -n monitoring
```

The main configuration file in the working tree is `monitoring/values.yaml`. Before running `helm upgrade`, inspect the existing release and values; an upgrade can change live resources.

If setting up a fresh environment, use the chart and values appropriate to your checked-out configuration. Do not assume that an example command or chart version is interchangeable with the deployed release.

### 7. Verify Catalogue metrics and alerting

Check the Catalogue service and endpoints:

```bash
kubectl get svc catalogue -n robot-shop
kubectl get endpoints catalogue -n robot-shop
kubectl get prometheusrule -n monitoring
```

The `CatalogueServiceDown` alert is intended to detect a missing or down Catalogue scrape target. Prometheus target health can be checked with:

```promql
up{namespace="robot-shop", service="catalogue"}
```

A value of `1` means the target was successfully scraped at that evaluation time; `0` means the target was discovered but the scrape failed. An absent result may mean the target is not being discovered or the series is absent.

### 8. Verify centralized logging

The repository includes configuration files for Loki and Fluent Bit:

```text
monitoring/loki-values.yaml
monitoring/fluent-bit-values.yaml
```

Check the deployed resources:

```bash
kubectl get pods,services,daemonsets -n monitoring
```

Confirm the Loki endpoint and Fluent Bit output configuration from the actual values files before applying or upgrading charts. In Grafana, select the configured Loki data source and use **Explore** to query logs.

### 9. Verify distributed tracing

Inspect the tracing manifests and deployed resources:

```bash
find tracing -maxdepth 2 -type f
kubectl get pods,services -n monitoring
```

Jaeger can display traces only when requests are instrumented and telemetry is successfully exported to the configured collector/backend. Confirm the actual service name and port before starting a port-forward.

### 10. Verify Slack alert notifications

The Slack notification path was tested successfully during this project. The Alertmanager configuration uses the Kubernetes Secret `slack-webhook` and sends matching alerts to Slack channel `#alerts`.

Check the relevant resources without printing secret contents:

```bash
kubectl get secret slack-webhook -n monitoring
kubectl get alertmanagerconfig -n monitoring
kubectl get prometheusrule -n monitoring
```

Never commit a webhook URL, display it in terminal output, or include it in screenshots. If a webhook is exposed, revoke it and create a replacement.

The controlled `SlackDeliveryTest` alert was received in Slack. The production `CatalogueServiceDown` rule was also updated with the namespace label required by the configured Alertmanager route. A notification from the test alert confirms the delivery path; it does not mean the Catalogue service is currently down.

---

## 💡 Useful Port-Forward Commands

Run a command in its own terminal and leave it running while using the local UI. First confirm the service name with `kubectl get services -n monitoring`; names can differ by Helm values.

### Prometheus

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open <http://localhost:9090>.

### Alertmanager

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

Open <http://localhost:9093>.

### Grafana

Find the actual Grafana service name:

```bash
kubectl get services -n monitoring
```

Then forward its service port to local port `3000`. For example, if the service is `monitoring-grafana` and exposes port `80`:

```bash
kubectl port-forward -n monitoring \
  svc/monitoring-grafana 3000:80
```

Open <http://localhost:3000> and use the credentials configured for your deployment. Do not place passwords in this README.

### Jaeger

Check the actual Jaeger service name and exposed UI port before forwarding:

```bash
kubectl get services -n monitoring
```

Then use the matching service and port from your deployment. Do not assume the service is named `jaeger` unless that is what the cluster reports.

### General health checks

```bash
kubectl get pods -n robot-shop
kubectl get services -n robot-shop
kubectl get pods -n monitoring
```

To inspect Catalogue logs:

```bash
kubectl logs -n robot-shop deployment/catalogue
```

---

## 🎯 Availability SLO and Error Budget

The documented Catalogue availability target is **99.9% over a rolling 30-day window**.

| Measurement | Value |
|---|---:|
| Availability objective | 99.9% |
| Allowed unavailability | 0.1% |
| Minutes in a 30-day window | 43,200 |
| Error budget at 99.9% | 43.2 minutes |

The error-budget policy documents how to respond when availability declines or the budget is exhausted.

**Current limitation:** the Catalogue alert detects scrape failures, but a validated rolling 30-day availability calculation and error-budget consumption metric have not yet been implemented. Treat 99.9% as a target, not a measured result.

---

## 🧪 Verification Checklist

- [ ] EKS cluster and worker nodes are healthy.
- [ ] Robot Shop pods and services are ready.
- [ ] Prometheus discovers and successfully scrapes the Catalogue target.
- [ ] Grafana can query its configured data sources.
- [ ] Loki receives logs from Fluent Bit.
- [ ] Jaeger displays traces from instrumented requests.
- [ ] Prometheus loads the `CatalogueServiceDown` alert rule.
- [x] Alertmanager configuration routes the test alert to the Slack receiver.
- [x] A controlled Slack test notification was received.
- [ ] Rolling 30-day SLO measurement is implemented and validated before reporting compliance.

The checked items reflect the work confirmed during this troubleshooting session; other items should be checked only after verifying them in the target environment.

---

## 🔐 Security and Cost Management

- **Credentials:** Use approved AWS authentication and Kubernetes Secrets for sensitive values.
- **Git hygiene:** Do not commit webhook URLs, local credentials, Terraform state, or generated files that contain secrets.
- **Infrastructure costs:** EKS, worker nodes, storage, load balancers, and telemetry retention can incur ongoing charges.
- **Resource usage:** Set appropriate resource requests, limits, retention, and persistence.
- **Production readiness:** Review access control, network exposure, encryption, backups, and alert coverage before production use.

---

## 🧹 Cleanup

> **Warning:** Cleanup can permanently remove workloads, data, storage, and infrastructure. Confirm the AWS account, cluster, namespace, and resource ownership before proceeding.

Inspect releases first:

```bash
helm list -A
kubectl get pvc -A
```

Only uninstall a release if you know it belongs to this project and is no longer needed. For example:

```bash
helm uninstall robot-shop -n robot-shop
helm uninstall monitoring -n monitoring
```

Uninstall Loki separately only if it was installed as its own Helm release. Removing Helm releases does not necessarily remove persistent volumes or every cloud resource.

If Terraform manages the infrastructure, inspect the configuration and destroy plan from the correct Terraform directory:

```bash
cd terraform-eks
terraform plan -destroy
```

Run `terraform destroy` only if you intentionally want to delete the Terraform-managed environment and have verified the plan:

```bash
terraform destroy
```

Review AWS resources afterward; resources created outside Terraform may need separate cleanup.

---

## 📚 Project Documentation

- [Runbook](monitoring/docs/runbook.md)
- [Postmortem](monitoring/docs/postmortem.md)

These links assume the files exist at the listed paths in the repository.
