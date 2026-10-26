
# Robot Shop Catalogue Incident Runbook

## Alert
`CatalogueServiceDown`

## Symptoms
The Catalogue service is unavailable or Prometheus cannot scrape it.

## Immediate checks
1. Check Catalogue pods:
   `kubectl get pods -n robot-shop -l service=catalogue`
2. Check the deployment:
   `kubectl get deployment catalogue -n robot-shop`
3. Check recent logs:
   `kubectl logs -n robot-shop deployment/catalogue --tail=100`
4. Check events:
   `kubectl get events -n robot-shop --sort-by=.lastTimestamp`

## Mitigation
If Catalogue was unintentionally scaled down, restore one replica:

`kubectl scale deployment catalogue -n robot-shop --replicas=1`

Wait for recovery:

`kubectl rollout status deployment/catalogue -n robot-shop --timeout=120s`

## Verify recovery
Confirm the pod is Ready and the health endpoint returns HTTP 200.

## Investigation
- Review Catalogue logs and Kubernetes events.
- Check the Catalogue target in Prometheus.
- Search relevant logs in Grafana/Loki.
- Inspect a related request trace in Jaeger.

## Escalation
If recovery fails, preserve logs and events and notify the project instructor.
