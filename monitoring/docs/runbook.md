
# Catalogue Service Incident Runbook

## Purpose
This runbook guides responders through diagnosing and recovering from a Catalogue service outage in Robot Shop.

## Alert
- **Alert name:** CatalogueServiceDown
- **Severity:** Critical
- **Trigger:** Prometheus cannot scrape the Catalogue service, or the target is missing, for more than two minutes.

## 1. Check the alert
Open the Prometheus Alerts page and confirm whether `CatalogueServiceDown` is firing.

## 2. Check Kubernetes
Run:

```powershell
kubectl get pods -n robot-shop
kubectl get deployment catalogue -n robot-shop
kubectl get svc catalogue -n robot-shop
kubectl get endpoints catalogue -n robot-shop
```

Look for unavailable replicas, restarting pods, or missing endpoints.

## 3. Inspect Catalogue logs
Run:

```powershell
kubectl logs -n robot-shop deployment/catalogue --tail=100
```

Look for application errors, database connection failures, or repeated restarts.

## 4. Check service health
Run:

```powershell
kubectl port-forward -n robot-shop svc/catalogue 8080:8080
```

In a second terminal:

```powershell
curl.exe http://127.0.0.1:8080/health
```

A healthy response should return HTTP 200 and indicate the application and MongoDB are healthy.

## 5. Recover the service
If the deployment has zero replicas and this is not an intentional maintenance activity:

```powershell
kubectl scale deployment/catalogue -n robot-shop --replicas=1
kubectl rollout status deployment/catalogue -n robot-shop
kubectl get pods -n robot-shop
```

Do not delete resources or restart dependencies without investigating the cause first.

## 6. Verify recovery
- Confirm the Catalogue pod is Running and Ready.
- Confirm the health endpoint returns HTTP 200.
- Confirm Prometheus reports the Catalogue target as up.
- Confirm the alert returns to a resolved state.
- Check whether the expected Slack notification was delivered.

## 7. Escalation and follow-up
If recovery fails, preserve relevant logs and events, investigate MongoDB connectivity and recent changes, and escalate to the project maintainer.

Document the incident timeline, impact, root cause, resolution, and follow-up actions in `monitoring/docs/postmortem.md`.

## Safety
Never put credentials, webhook URLs, or other secrets in this runbook.
