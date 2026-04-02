---
title: "Building an Alerting Strategy That Doesn't Burn Out Your Team"
description: "Alert fatigue kills reliability. Learn how to design an alerting system with the right thresholds, routing, and escalation policies."
pubDate: '2026-03-10'
tags: ['alerting', 'sre', 'on-call']
author: 'Jordan Park'
---

# Building an Alerting Strategy That Doesn't Burn Out Your Team

Alert fatigue is one of the leading causes of on-call burnout. When every alert is noise, engineers stop responding to alerts — including the real ones. This post covers how to design an alerting strategy that pages people only when humans actually need to act.

## The Core Principle: Alerts Should Require Human Action

Every alert should answer "yes" to all three questions:

1. **Is something actually broken right now?** (Not "might break soon")
2. **Is it urgent enough to wake someone up?**
3. **Is there a specific action a human must take?**

If an alert fires but the system recovers by itself, that's a warning to log, not an alert to page. Auto-recovering systems should log the recovery and let you review it in the morning.

## The Alert Taxonomy

Not all alerts are equal. Design your alerting system with multiple severity levels:

### P0 — Critical (page immediately, 24/7)
Revenue is affected right now. User data is at risk. Core services are down.

```yaml
- name: payment-service-down
  condition: error_rate > 50% for 2 minutes
  severity: p0
  channels:
    - pagerduty: escalation-policy-primary
  message: "Payment service error rate is {{error_rate}}%. Immediate action required."
```

### P1 — High (page during business hours, overnight for on-call)
Significant degradation, but the system is limping along.

```yaml
- name: slow-checkouts
  condition: p95_latency > 5s for 5 minutes
  severity: p1
  channels:
    - pagerduty: escalation-policy-secondary
    - slack: "#incidents"
```

### P2 — Medium (Slack notification, no page)
Something to investigate but not urgent.

```yaml
- name: high-queue-depth
  condition: job_queue_depth > 10000
  severity: p2
  channels:
    - slack: "#ops-monitoring"
```

### P3 — Low (logged, reviewed weekly)
Trends worth watching.

## Alert on Symptoms, Not Causes

A common mistake is alerting on causes (CPU > 80%, memory > 90%) instead of symptoms (users experiencing errors). Causes are often noisy and frequently resolve themselves. Symptoms directly measure user experience.

**Noisy (cause-based):**
```yaml
- name: high-cpu
  condition: cpu_usage > 80%
  # Fires constantly on normal traffic spikes
```

**Better (symptom-based):**
```yaml
- name: api-error-rate
  condition: http_5xx_rate > 1% for 3 minutes
  # Only fires when users are actually experiencing errors
```

## The Four Golden Signals

Google's SRE book defines four golden signals that cover most failure modes:

1. **Latency** — How long do requests take? Alert on P95/P99.
2. **Traffic** — How many requests? Sudden drops can indicate failures.
3. **Errors** — What percentage of requests are failing?
4. **Saturation** — How full is the system? Queue depth, connection pool usage.

Alert on these before adding anything else.

## Job Monitoring Alerts

For scheduled jobs and background workers, apply the dead-man switch pattern:

```yaml
jobs:
  - name: nightly-billing-run
    schedule: "0 1 * * *"
    expected_duration: 15m
    grace_period: 30m  # Alert if not completed 30m after expected start
    
    alert_on:
      - miss:           # Job didn't start
          severity: p0
          message: "Billing job missed its schedule. Customers will not be charged."
      
      - failure:        # Non-zero exit code
          severity: p0
          message: "Billing job failed with exit code {{exit_code}}."
      
      - duration_gt: 45m  # Job is hanging
          severity: p1
          message: "Billing job has been running for {{duration}}. Expected ~15m."
      
      - duration_lt: 2m  # Job finished suspiciously fast (processed nothing?)
          severity: p1
          message: "Billing job completed in {{duration}} — may have processed no records."
```

The last alert — on suspiciously short duration — catches silent zero-record failures that dead-man switches alone miss.

## Escalation Policies

Even well-designed alerts can be missed. Always configure escalation:

```
On-call engineer notified
    ↓ (no acknowledgement in 10 minutes)
Secondary on-call notified
    ↓ (no acknowledgement in 10 minutes)
Engineering manager notified
    ↓ (no acknowledgement in 15 minutes)
VP Engineering notified
```

## Runbooks: Every Alert Needs One

Every alert should link to a runbook — a short document that tells the on-call engineer what to check and how to resolve the issue. Even a 3-step runbook is better than nothing.

```yaml
- name: database-connection-pool-exhausted
  condition: db_connections_available < 5
  severity: p1
  runbook: "https://wiki.internal/runbooks/db-connection-pool"
  message: |
    DB connection pool is exhausted ({{connections_available}} remaining).
    See runbook for remediation steps.
```

## Review Your Alerts Monthly

Schedule a monthly alert review:
- Which alerts fired most often? Are they noisy?
- Which alerts were never acknowledged within 5 minutes? Too many?
- Which incidents happened with no alert? Add coverage.

Alerting is not a set-and-forget system. It requires the same ongoing maintenance as your code.

## Conclusion

Great alerting is less about tools and more about discipline. Alert only on symptoms. Design clear severity levels. Write runbooks. Review regularly. Your on-call team will be more focused, less burned out, and more effective when it actually matters.
