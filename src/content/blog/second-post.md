---
title: 'Structured Logging Best Practices for Production Systems'
description: 'Unstructured logs are a liability. Learn how to design log schemas, use correlation IDs, and make your logs queryable at scale.'
pubDate: '2026-02-14'
tags: ['logging', 'observability', 'devops']
author: 'Alex Thornton'
---

# Structured Logging Best Practices for Production Systems

At 3 AM during an incident, the quality of your logs is the difference between a 10-minute fix and a 3-hour debugging session. Unstructured text logs force you to write regex to extract information that should have been structured from the start. This post covers how to design logs that are useful under pressure.

## What Is Structured Logging?

Instead of writing:

```
[2026-02-14 09:12:03] ERROR: Payment failed for order 8821 — Stripe timeout after 5000ms
```

You write:

```json
{
  "timestamp": "2026-02-14T09:12:03.412Z",
  "level": "error",
  "service": "payment-service",
  "event": "payment.failed",
  "order_id": "8821",
  "provider": "stripe",
  "error": "timeout",
  "duration_ms": 5000,
  "trace_id": "4bf9a2c1-7e3d-4a12-9b55-e8c3f0d1a482"
}
```

Now you can query: `event:payment.failed provider:stripe duration_ms:>3000` and get exactly the slow Stripe failures.

## Essential Fields

Every log entry should include these fields:

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO 8601 | When it happened (UTC always) |
| `level` | enum | `debug`, `info`, `warn`, `error`, `fatal` |
| `service` | string | Which service emitted this log |
| `trace_id` | string | Request correlation ID |
| `event` | string | Dot-notation event name |
| `message` | string | Human-readable summary |

```python
import structlog
import uuid

log = structlog.get_logger()

# Bind context that applies to all logs in this request
request_log = log.bind(
    service="payment-service",
    trace_id=str(uuid.uuid4()),
    user_id=user.id,
)

request_log.info("payment.initiated", order_id=order.id, amount=order.total)
```

## Correlation IDs

A correlation ID (also called a trace ID) lets you group all log events from a single user request across multiple services. Generate it at the edge and propagate it through every downstream call.

```python
# In your HTTP middleware
import uuid
from fastapi import Request

async def correlation_middleware(request: Request, call_next):
    trace_id = request.headers.get("X-Trace-ID") or str(uuid.uuid4())
    
    # Store in context var so all logs in this request use it
    with structlog.contextvars.bind_contextvars(trace_id=trace_id):
        response = await call_next(request)
        response.headers["X-Trace-ID"] = trace_id
        return response
```

Now when you search LogPulse for `trace_id:4bf9a2c1`, you see every log event across your entire distributed system for that single user request.

## Log Levels: Use Them Correctly

Teams that log everything at `INFO` create signal-to-noise problems. Use levels deliberately:

- **DEBUG**: Verbose details useful only during development. Disabled in production.
- **INFO**: Normal operational events. Job started, user logged in, payment processed.
- **WARN**: Something unexpected happened but the system recovered. Retry succeeded, fallback used.
- **ERROR**: A failure that requires investigation. A payment failed, a job crashed.
- **FATAL**: The process cannot continue. Used just before `os.exit()`.

A useful rule: if you wouldn't want to be paged for it, it's not an ERROR.

## Sampling High-Volume Logs

Some events (like HTTP access logs) occur thousands of times per second. Logging every one is expensive and creates noise. Use **sampling** to keep a representative subset.

```python
import random

def should_sample(level: str, sample_rate: float = 0.01) -> bool:
    if level in ("error", "fatal", "warn"):
        return True  # Always log errors
    return random.random() < sample_rate  # Sample 1% of info/debug

if should_sample(level):
    logger.log(level, message, **fields)
```

Always sample 100% of errors — you never want to lose a failure signal.

## Sensitive Data

Never log passwords, tokens, credit card numbers, or PII. Scrub before logging:

```python
SENSITIVE_KEYS = {"password", "token", "secret", "card_number", "ssn"}

def scrub(data: dict) -> dict:
    return {
        k: "***REDACTED***" if k in SENSITIVE_KEYS else v
        for k, v in data.items()
    }

logger.info("user.updated", **scrub(user_data))
```

## Conclusion

Structured logging is a force multiplier for your on-call team. The 30 minutes you invest in designing your log schema today will save hours during your next incident. Consistency is key — define a schema, enforce it with a shared logger class, and make it easy for every developer to follow the same patterns.
