# Robot Shop Incident Postmortem

## Incident Summary
The Catalogue service was intentionally scaled to zero replicas
during a controlled incident drill.

## Impact
Catalogue became temporarily unavailable during the drill.
The incident was planned and used to validate alerting and recovery.

## Detection
Prometheus raised the CatalogueServiceDown alert after the
configured two-minute evaluation period.

## Timeline
- Before drill: Catalogue was healthy and Prometheus reported up = 1.
- Incident initiated: Catalogue deployment scaled to zero replicas.
- Detection: CatalogueServiceDown entered the Firing state.
- Mitigation: Catalogue deployment restored to one replica.
- Recovery: Catalogue pod returned to Running and Ready.

## Root Cause
The Catalogue deployment had zero replicas because of the
intentional scale-down performed during the drill.

## Resolution
The deployment was scaled back to one replica. The Catalogue
pod returned to a healthy state.

## Lessons Learned
- Alerts should detect both failed scrapes and missing targets.
- Recovery should be verified using Kubernetes and the health endpoint.
- Runbook links should be included in alert annotations.

## Follow-up Actions
- Verify Slack notification delivery.
- Confirm logs are searchable by service and severity.
- Verify the runbook link from the alert.
- Export the Grafana dashboard as JSON.

## Blameless Statement
The drill exposed opportunities to improve monitoring and incident
response. Follow-up actions focus on system reliability and
clearer operational procedures, not individual blame.