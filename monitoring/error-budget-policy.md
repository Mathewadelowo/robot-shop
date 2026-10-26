# Robot Shop Availability SLO and Error Budget Policy

## 1. Service objective
Service: Robot Shop Catalogue API

SLO: 99.9% availability over a rolling 30-day window.

Availability is measured using successful Prometheus scrapes of the Catalogue service as a practical proxy for service availability. This proxy does not guarantee that every user request succeeds.

## 2. Error budget
A 99.9% availability target permits 0.1% unavailability.

For a 30-day period:
- Total time: 43,200 minutes
- Error budget: approximately 43.2 minutes
- Availability below 99.9% means the SLO has been missed.

## 3. Monitoring and alerting
Prometheus monitors the Catalogue service.
The CatalogueServiceDown alert fires when Prometheus cannot scrape the service for more than two minutes.
Alertmanager routes this alert to the Slack #alerts channel.

## 4. Error budget policy
- Healthy budget: continue planned releases while following normal testing and rollback practices.
- Budget running low: review recent incidents, increase monitoring, and prioritize reliability fixes.
- Budget exhausted: pause non-essential feature releases until the team reviews the causes and agrees on corrective actions.
- Critical security fixes and changes needed to restore service may proceed with appropriate safeguards.

## 5. Incident response
When an availability alert fires, the responsible operator acknowledges it, investigates the service and its dependencies, restores service, and records the incident and downtime.

## 6. Review
Review availability and error-budget consumption at least monthly. Reassess the SLO if measurement quality or service expectations change.

Note: The 99.9% target is the policy objective. A rolling 30-day availability calculation must be implemented and validated before reporting measured SLO compliance.
