# Recent Work Summary

Date: 2026-10-24

## Overview

Over the last working session, the focus was on making the deployment configuration more portable and easier to manage. The main thread was removing hard-coded monitoring and platform assumptions from the Helm chart, while also starting a Terraform-based EKS path for cloud infrastructure provisioning.

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



## Why these changes were made

The common theme was flexibility and portability.

- In `K8s/helm/templates/cart-deployment.yaml`, `catalogue-deployment.yaml`, `dispatch-deployment.yaml`, `payment-deployment.yaml`, `user-deployment.yaml`, and `web-deployment.yaml`, environment-specific Instana agent host and EUM settings were removed or made conditional. The corresponding EUM values were removed from `K8s/helm/values.yaml`. This avoids requiring monitoring configuration that may not exist in every cluster and makes the chart easier to reuse across environments. Docker Compose EUM configuration is separate and was not changed.
- In `K8s/helm/templates/web-deployment.yaml`, the web workload's service-account and PSP wiring was removed. This avoids coupling the web deployment to chart-specific PSP/service-account setup when that setup is not needed.
- In `K8s/helm/templates/web-service.yaml`, the default Service type changed from `LoadBalancer` to `ClusterIP`. This avoids requesting an external load balancer by default and leaves exposure to the deployment's chosen ingress, platform Route, or explicit NodePort setting.
- In `K8s/helm/templates/redis-statefulset.yaml`, Redis's data volume uses `emptyDir` rather than a persistent volume claim. This was chosen because the target cluster does not have the EBS CSI driver needed to provision EBS-backed persistent storage. The trade-off is intentional: the Redis data is ephemeral and is lost when the pod restarts.
- The Terraform files in `terraform-eks/` (`eks.tf`, `iam.tf`, `local.tf`, `outputs.tf`, `providers.tf`, and `variables.tf`) add an independent AWS EKS provisioning path. This lets the application run on an AWS-managed Kubernetes cluster instead of relying only on the existing static manifests or Helm deployment assumptions. The configuration creates its networking, IAM roles, cluster, and managed node group so those infrastructure resources can be managed together with Terraform.







.
