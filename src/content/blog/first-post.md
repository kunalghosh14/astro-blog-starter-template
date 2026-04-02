---
title: 'How to Monitor Cron Jobs Without Losing Sleep'
description: 'Cron jobs fail silently. Learn how dead-man switch monitoring, structured logging, and smart alerting keep your scheduled tasks reliable.'
pubDate: '2026-01-20'
tags: ['monitoring', 'cron', 'reliability']
author: 'Lena Kovacs'
---

# How to Monitor Cron Jobs Without Losing Sleep

Cron jobs are the unsung workhorses of every production system. They send invoices, sync databases, generate reports, clean up stale data. And when they fail, they usually fail silently — no exceptions bubble up, no alerts fire, no dashboards turn red. You find out three days later when a customer asks why their weekly report never arrived.

This post covers the practical patterns for making cron job failures impossible to miss.

## The Problem: Silent Failures

A typical cron entry looks like this:

```bash
0 2 * * * /usr/bin/python3 /opt/jobs/daily_report.py >> /var/log/daily_report.log 2>&1
```

At first glance, this looks fine. It runs at 2 AM, logs output. But consider these failure modes:

- The job exits with code 0 but produces empty output (logic error)
- The job starts but hangs on a database lock (never finishes)
- The server is overloaded and the job doesn't start at all
- The log file fills the disk and `>>` silently truncates

`cron` will email the output to the local `root` user, which no one reads. You are flying blind.

## Pattern 1: Dead-Man Switch Monitoring

The most reliable pattern is a **dead-man switch** (also called a heartbeat or check-in). The idea is simple: your job sends a "ping" to a monitoring endpoint on success. If the ping doesn't arrive within a defined window, you get alerted.

```python
import requests
import sys

PING_URL = "https://ping.logpulse.io/your-job-id"

def run():
    # ... your job logic ...
    return True

if __name__ == "__main__":
    try:
        success = run()
        if success:
            requests.get(f"{PING_URL}/succeed", timeout=10)
        else:
            requests.get(f"{PING_URL}/fail", timeout=10)
    except Exception as e:
        requests.get(f"{PING_URL}/fail", params={"msg": str(e)}, timeout=10)
        sys.exit(1)
```

This catches almost all failure modes:
- Logic failures (call `/fail` explicitly)
- Crashes (call `/fail` in the except block)
- Hangs (the ping never arrives, timeout alert fires)
- Server failures (the ping never arrives)

## Pattern 2: Structured Logging

Plain text logs are hard to query and alert on. Switch to structured JSON logging so your monitoring platform can parse fields automatically.

```python
import json, sys, time
from datetime import datetime, timezone

class StructuredLogger:
    def __init__(self, job_name: str):
        self.job_name = job_name
        self.start_time = time.time()

    def log(self, level: str, message: str, **fields):
        entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": level,
            "job": self.job_name,
            "message": message,
            "elapsed_ms": round((time.time() - self.start_time) * 1000),
            **fields
        }
        print(json.dumps(entry))

logger = StructuredLogger("daily-report")
logger.log("info", "job started", rows_expected=50000)
```

Now your monitoring platform can alert on `level == "error"`, graph `elapsed_ms`, or detect jobs that processed zero rows.

## Pattern 3: Wrap With a CLI Tool

For maximum coverage with minimal code changes, wrap your job with a monitoring CLI:

```bash
# Before
0 2 * * * /usr/bin/python3 /opt/jobs/daily_report.py

# After
0 2 * * * logpulse exec --job daily-report -- /usr/bin/python3 /opt/jobs/daily_report.py
```

The wrapper captures stdout/stderr, records duration, detects non-zero exit codes, and fires alerts automatically.

## Setting Alert Windows

Configure your alert window based on expected job duration. If your daily report usually takes 8 minutes, set a grace period of 20 minutes.

```yaml
jobs:
  - name: daily-report
    schedule: "0 2 * * *"
    grace_period: 20m
    alert_on:
      - miss
      - failure
      - duration_gt: 30m
    channels:
      - slack: "#ops-alerts"
      - pagerduty
```

## Conclusion

Cron jobs deserve the same observability as your HTTP endpoints. Dead-man switch monitoring catches the failure modes that try/except blocks can't — hangs, missed runs, and total server failures. Your future on-call self will thank you.
