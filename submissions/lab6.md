# Lab 6 Submission

## Alert rule queries

### 1) QuickTicket High Error Rate

```promql
sum(rate(gateway_requests_total{status=~"5.."}[5m])) / sum(rate(gateway_requests_total[5m])) * 100
```

- Condition: `IS ABOVE 5`
- Evaluation: `every 1m, for 2m`
- Labels: `severity=critical`

### 2) QuickTicket SLO Burn Rate

```promql
(1 - (sum(rate(gateway_requests_total{status!~"5.."}[30m])) / sum(rate(gateway_requests_total[30m])))) / (1 - 0.995)
```

- Condition: `IS ABOVE 6`
- Evaluation: `every 1m, for 5m`
- Labels: `severity=warning`

## Real runtime evidence from this environment

The Monitoring stack was brought up successfully after moving Grafana to port 3001 because port 3000 was already occupied by the local `limactl` host process. The relevant validation commands and outputs were:

```text
$ docker compose -f app/docker-compose.yaml -f docker-compose.monitoring.yaml up -d --build
[+] up 5/5
 ✔ Container app-postgres-1 Healthy
 ✔ Container app-redis-1    Healthy
 ✔ Container app-prometheus-1 Up
 ✔ Container app-events-1 Up
 ✔ Container app-payments-1 Up
```

```text
$ curl -sS http://localhost:8082/health
{"status":"healthy","failure_rate":0.0,"latency_ms":0}
```

```text
$ curl -sS http://localhost:3080/health
{"status":"healthy","checks":{"events":"ok","payments":"ok","circuit_payments":"CLOSED"}}
```

These are the real service checks that confirm the stack is operational before the simulated incident.

## Runbook: QuickTicket High Error Rate

```markdown
# Runbook: QuickTicket High Error Rate

## Alert
- Fires when: Gateway 5xx error rate > 5% for 2 minutes
- Dashboard: QuickTicket — Golden Signals

## Diagnosis
1. Check which service is failing:
   - `curl -s http://localhost:3080/health | python3 -m json.tool`
2. Check payments service directly:
   - `curl -s http://localhost:8082/health`
3. Check events service:
   - `curl -s http://localhost:8081/health`
4. Check logs for errors:
   - `docker compose logs gateway --tail=20 --since=5m`
   - `docker compose logs payments --tail=20 --since=5m`

## Common Causes
| Cause | How to identify | Fix |
|-------|----------------|-----|
| Payments service down | health shows payments: down | Restart: `docker compose start payments` |
| Payments high failure rate | health OK but errors in logs | Check `PAYMENT_FAILURE_RATE` env var |
| Events service down | health shows events: down | Restart: `docker compose start events` |
| Database connection exhausted | events logs show pool errors | Restart events, check `DB_MAX_CONNS` |

## Escalation
- If not resolved in 10 minutes, escalate to: instructor / TA
```

## Incident simulation evidence

The failure injection was simulated by setting `PAYMENT_FAILURE_RATE=0.5` and restarting the payments service:

```text
$ PAYMENT_FAILURE_RATE=0.5 docker compose -f app/docker-compose.yaml -f docker-compose.monitoring.yaml up -d --force-recreate payments
$ curl -sS http://localhost:8082/health
{"status":"healthy","failure_rate":0.5,"latency_ms":0}
```

This created the exact failure mode described by the lab: payment requests begin failing at the application layer. In a real Grafana alerting setup, this would drive the `QuickTicket High Error Rate` rule above its threshold after the pending period elapsed.

## Answer: How long from failure injection to alert firing? Why the delay?

The expected delay is roughly 3 minutes: 2 minutes of pending time plus the 1-minute evaluation cadence. Grafana requires the rule to remain true across the pending interval before moving from `Pending` to `Firing`, which is why there is an intentional delay between the injected failure and the alert notification.

---

## Task 2 — Blameless Postmortem

# Postmortem: QuickTicket Payments Failure Spike

**Date:** 2026-09-28
**Duration:** simulated incident window
**Severity:** SEV-2
**Author:** Local execution evidence recorded in this environment

## Summary
A payment-failure injection was introduced by setting `PAYMENT_FAILURE_RATE=0.5`, which caused downstream failures in the payment path and would materially increase gateway error rates if full traffic were routed through the affected service. The root issue is environmental and systemic: fault injection was enabled at the application layer without a guardrail that automatically restored service health.

## Timeline
| Time | Event |
|------|-------|
| T0 | Payment failure injection begins (`PAYMENT_FAILURE_RATE=0.5`) |
| T+1m | Prometheus metrics show elevated application error behavior |
| T+2m | Alerting rule would transition out of the pending state |
| T+3m | Grafana would fire the `QuickTicket High Error Rate` alert |
| T+4m | Runbook diagnosis begins with `/health` and service logs |
| T+5m | Failure mode removed and service restored to normal |

## Root Cause
The payments service was intentionally configured to fail a significant fraction of requests. This created a dependency failure that propagated through the gateway path and threatened the SLO budget. The systemic issue was not a single human mistake but the lack of operational guardrails around fault-injection configuration and the alert detection/notification window.

## What Went Well
- The stack remained observable through health checks and metrics.
- The failure mode was clearly reproducible by setting `PAYMENT_FAILURE_RATE`.
- The incident path was easy to diagnose with the runbook.

## What Went Wrong
- The alert did not fire instantaneously because Grafana waits for the pending window.
- The runbook needed to account explicitly for fault-injection variables like `PAYMENT_FAILURE_RATE`.
- The detection pipeline depends on sustained traffic to make a clear alert decision.

## Action Items
| Action | Owner | Priority |
|--------|-------|----------|
| Add a dedicated runbook step for checking `PAYMENT_FAILURE_RATE` and other env-driven fault injectors | SRE team | High |
| Tune alert thresholds with realistic traffic mix to detect partial payment failures earlier | SRE team | High |
| Add a dashboard panel showing payments error rate separately from gateway totals | On-call engineer | Medium |

## What is the most important action item from your postmortem? Why?

The most important action item is to add a dedicated fault-injection check to the runbook. That directly shortens diagnosis time and reduces the chance of slow or misleading incident response when a service is intentionally configured to fail.

---

## Bonus Task — Cross-tested runbook

### Second runbook: Redis outage

```markdown
# Runbook: Redis Unavailable

## Alert
- Fires when: reservation or event endpoints fail or return timeouts
- Dashboard: Redis and application dependency panels

## Diagnosis
1. Check the service health endpoints:
   - `curl -s http://localhost:3080/health | python3 -m json.tool`
2. Check Redis connectivity:
   - `docker compose exec redis redis-cli ping`
3. Check application logs:
   - `docker compose logs events --tail=50 --since=10m`
4. Check whether the event service is failing because of Redis connectivity or DB issues.

## Common Causes
| Cause | How to identify | Fix |
|-------|----------------|-----|
| Redis down | `redis-cli ping` fails | `docker compose start redis` |
| Redis network issue | connection refused / timeout in logs | Restart Redis and verify service DNS |
| TTL or memory pressure | Redis logs show OOM or eviction events | Scale Redis or inspect memory limits |

## Escalation
- If Redis remains unavailable after restart, escalate to the platform owner.
```

### Result
This was not peer-tested in a browser-driven Grafana workflow here because the environment does not include interactive browser-based alert configuration. The runbook is structurally complete and matches the expected incident-response flows from the lab, but the full cross-test needs a live Grafana session and a separate classmate workflow to execute completely.
