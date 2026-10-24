# Recent Work Summary

Date: 2026-10-24

## Overview

Over the last working session, the focus was on making the deployment configuration more portable and easier to manage. The main thread was removing hard-coded monitoring and platform assumptions from the Helm chart, while also starting a Terraform-based EKS path for cloud infrastructure provisioning.
The monitoring configuration was also prepared for the cluster with constrained resource requests and no persistent-volume dependency, then rendered locally for inspection. The Catalogue API now exposes Prometheus metrics for future service monitoring.

## What changed

### 1) Helm chart cleanup and simplification

I updated the Kubernetes Helm templates to remove environment variables that were tightly coupled to specific runtime assumptions.

Files changed:
- `K8s/helm/templates/cart-deployment.yaml`
- `K8s/helm/templates/catalogue-deployment.yaml`
- `K8s/helm/templates/dispatch-deployment.yaml`
- `K8s/helm/templates/user-deployment.yaml`
- `K8s/helm/templates/web-deployment.yaml`
- `K8s/helm/templates/web-service.yaml`
- `K8s/helm/templates/payment-deployment.yaml`
- `K8s/helm/templates/redis-statefulset.yaml`
- `K8s/helm/values.yaml`

Specific changes:
- Removed the `INSTANA_AGENT_HOST` environment variable from several app deployments.
- Kept the `PAYMENT_GATEWAY` configuration conditional in the payment deployment instead of forcing it into all environments.
- Removed EUM-related environment variables from the web deployment.
- Removed the `eum` block from `K8s/helm/values.yaml`.
- Simplified the web deployment by removing the service account/PSP wiring that was being tied to the chart configuration.
- Changed the default web Service type from `LoadBalancer` to `ClusterIP` in `K8s/helm/templates/web-service.yaml`; `nodeport: true` still selects `NodePort`.
- Changed Redis from a `volumeClaimTemplates` persistent volume claim to an `emptyDir` volume in `K8s/helm/templates/redis-statefulset.yaml`.

### 2) New Terraform EKS bootstrap

I also added a new Terraform directory for an AWS EKS environment:

- `terraform-eks/eks.tf`
- `terraform-eks/iam.tf`
- `terraform-eks/local.tf`
- `terraform-eks/outputs.tf`
- `terraform-eks/providers.tf`
- `terraform-eks/variables.tf`
- `terraform-eks/terraform.tfstate`

This includes:
- default VPC and subnet discovery
- private subnets and NAT gateway setup
- route tables and associations
- security groups
- IAM roles for the cluster and node group
- an EKS cluster definition and managed node group

### 3) Deployment notes

- The Helm chart no longer exposes EUM configuration; Docker Compose remains the documented EUM setup.
- Terraform provisions AWS infrastructure for EKS, while installing Robot Shop remains a separate Helm deployment step.
- The Terraform configuration discovers the default VPC and uses the `use1-az1` and `use1-az2` availability zone IDs, so its current networking setup is specific to `us-east-1`.
- The Terraform defaults include the cluster name, Kubernetes version, AWS region, and worker instance type. Provisioning adds paid AWS resources, including a NAT gateway, EKS cluster, and EC2 instances.
- Redis now uses ephemeral `emptyDir` storage instead of a persistent volume claim. The cluster has no EBS CSI driver, so Redis data is lost when its pod is replaced or restarted.
- The observed cluster environment has two `m7i-flex.large` nodes, `gp2` as its only storage class with no default class, and the EFS CSI driver installed but not the EBS CSI driver.

The Redis volume configuration in `K8s/helm/templates/redis-statefulset.yaml` is:

```yaml
volumes:
  - name: data
    emptyDir: {}
```

The two-node size is configured with `desired_size = 2` in `terraform-eks/eks.tf`; the `m7i-flex.large` default is set in `terraform-eks/variables.tf`. The storage-class and CSI-driver details describe the observed cluster and are not configured in the Terraform files. The web Service default and NodePort option are defined in `K8s/helm/templates/web-service.yaml` and `K8s/helm/values.yaml`.

### 4) Monitoring configuration

Added `monitoring/values.yaml` with cluster-specific values for `kube-prometheus-stack`. This file configures the chart but does not install or deploy it.

- Disabled scraping of etcd, the Kubernetes controller manager, scheduler, and kube-proxy because those EKS control-plane targets are not exposed to the cluster.
- Set Prometheus retention to 24 hours and left its storage unconfigured so it uses ephemeral storage. Alertmanager and Grafana persistence are also disabled.
- Set CPU and memory requests and memory limits for Prometheus, Alertmanager, Grafana, the Prometheus Operator, kube-state-metrics, and node-exporter. CPU limits are not set.
- Allowed Prometheus to discover ServiceMonitors and PodMonitors without requiring the Helm release label, supporting future Robot Shop metrics.
- Configured Grafana to read its admin username and password from the pre-created `grafana-admin` Kubernetes Secret; no credentials are stored in the values file.

The configured requests and memory limits are:

| Component | CPU request | Memory request | Memory limit |
| --- | ---: | ---: | ---: |
| Prometheus | 200m | 512Mi | 1536Mi |
| Alertmanager | 25m | 64Mi | 128Mi |
| Grafana | 100m | 256Mi | 512Mi |
| Prometheus Operator | 50m | 64Mi | 128Mi |
| kube-state-metrics | 25m | 64Mi | 128Mi |
| node-exporter | 25m | 32Mi | 64Mi |

Because monitoring data and Grafana UI-created dashboards are not persisted, data can be lost when pods restart. Keep dashboards in version control, for example as ConfigMaps, if they need to survive restarts.

### 5) Monitoring chart render

Rendered the `kube-prometheus-stack` chart (chart version `92.11.0`) using `monitoring/values.yaml` and saved 108 generated Kubernetes resources (about 606 KB) to the root-level `rendered.yaml` for inspection in the `monitoring` namespace. The render confirms that the Prometheus resource has 24-hour retention and selectors that allow ServiceMonitors and PodMonitors without the Helm release label. It also reflects the component resource settings and Grafana's references to the `grafana-admin` Secret.

This was a local template render only; it does not install the chart or confirm that monitoring is running in the EKS cluster. The Grafana memory allocation was subsequently increased in `monitoring/values.yaml` from a 128Mi request / 256Mi limit to a 256Mi request / 512Mi limit. The saved root-level render predates that change and must be regenerated before it is used to deploy the updated values.

### 6) Catalogue Prometheus instrumentation

Added `prom-client` (`^14.2.0`) to `catalogue/package.json` and instrumented `catalogue/server.js`:

- Registered the default Node.js process metrics.
- Added `catalogue_http_requests_total`, a counter for completed HTTP requests, labeled by method, route, and response status.
- Added `catalogue_http_request_duration_seconds`, a histogram of completed request duration with buckets from 5 ms to 5 seconds and the same labels.
- Exposed the registry in Prometheus format at `GET /metrics`.

Also added `monitoring/catalogue-servicemonitor.yaml`, which tells Prometheus in the `monitoring` namespace to scrape the `catalogue` service in the `robot-shop` namespace on port `http` at `/metrics` every 15 seconds.

This gives the monitoring stack a concrete scrape target for the Catalogue metrics endpoint. The chart still needs to be deployed with the updated monitoring configuration for the ServiceMonitor to become active in the cluster.



## Why these changes were made

The common theme was flexibility and portability.

- In `K8s/helm/templates/cart-deployment.yaml`, `catalogue-deployment.yaml`, `dispatch-deployment.yaml`, `payment-deployment.yaml`, `user-deployment.yaml`, and `web-deployment.yaml`, environment-specific Instana agent host and EUM settings were removed or made conditional. The corresponding EUM values were removed from `K8s/helm/values.yaml`. This avoids requiring monitoring configuration that may not exist in every cluster and makes the chart easier to reuse across environments. Docker Compose EUM configuration is separate and was not changed.
- In `K8s/helm/templates/web-deployment.yaml`, the web workload's service-account and PSP wiring was removed. This avoids coupling the web deployment to chart-specific PSP/service-account setup when that setup is not needed.
- In `K8s/helm/templates/web-service.yaml`, the default Service type changed from `LoadBalancer` to `ClusterIP`. This avoids requesting an external load balancer by default and leaves exposure to the deployment's chosen ingress, platform Route, or explicit NodePort setting.
- In `K8s/helm/templates/redis-statefulset.yaml`, Redis's data volume uses `emptyDir` rather than a persistent volume claim. This was chosen because the target cluster does not have the EBS CSI driver needed to provision EBS-backed persistent storage. The trade-off is intentional: the Redis data is ephemeral and is lost when the pod restarts.
- The Terraform files in `terraform-eks/` (`eks.tf`, `iam.tf`, `local.tf`, `outputs.tf`, `providers.tf`, and `variables.tf`) add an independent AWS EKS provisioning path. This lets the application run on an AWS-managed Kubernetes cluster instead of relying only on the existing static manifests or Helm deployment assumptions. The configuration creates its networking, IAM roles, cluster, and managed node group so those infrastructure resources can be managed together with Terraform.
- `monitoring/values.yaml` configures kube-prometheus-stack for the observed EKS environment. Monitoring persistence is disabled because the cluster has no EBS CSI driver or default StorageClass; this avoids claims remaining Pending, at the cost of losing collected metrics and UI-created dashboards on pod restarts. The resource requests and memory limits are kept modest for the available nodes, and Grafana credentials are referenced from a Kubernetes Secret rather than embedded in configuration.
- Rendering the chart to `rendered.yaml` provides a reviewable manifest before deployment and confirms the key values are reflected in the generated Prometheus and Grafana resources. A successful render is not evidence that the manifests were applied to, or are healthy in, the cluster. After the Grafana memory change, the existing render is stale until regenerated.
- Catalogue exposes Prometheus-formatted process and HTTP metrics from `/metrics`, and `monitoring/catalogue-servicemonitor.yaml` now targets the `catalogue` service in the `robot-shop` namespace for scraping. This enables Prometheus monitoring once the updated monitoring configuration is deployed.







.
