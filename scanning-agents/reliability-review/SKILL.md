---
name: Reliability Review
description: >
  Analyzes error handling, failure modes, observability, and resilience patterns.
  Finds the gaps where errors are swallowed, retries are missing, timeouts don't exist,
  and failures cascade instead of being contained.
category: reliability
---

# Reliability Review

Analyze the codebase for reliability gaps — the places where things fail silently, fail
badly, or fail in ways that are invisible to operators. This is the difference between
"the service had a brief issue" and "we found out from a customer tweet 3 hours later."

## Analysis Strategy

### 1. Trace error propagation paths
- Start from entry points (HTTP handlers, queue consumers, scheduled jobs)
- For each, follow the call chain and identify every place an error can occur: DB calls, external APIs, file I/O, parsing, validation
- At each error site, check: is the error caught? Logged? Propagated? Transformed? Swallowed?
- Look for: catch blocks that do nothing, generic error handlers that lose context, errors converted to null/undefined

### 2. Map external dependencies
- Find every external call: databases, APIs, caches, queues, file systems, third-party services
- For each: is there a timeout? Retry logic? Circuit breaker? Fallback behavior?
- What happens when the dependency is slow (not down, just slow)?
- What happens when it returns unexpected data?

### 3. Assess observability
- What is logged? At what level? Is there enough context to diagnose a production issue?
- Are there request/correlation IDs flowing through the system?
- Are important business events tracked? (not just errors — things like "user completed checkout", "payment processed")
- Can you answer "what happened to request X?" from the logs alone?

### 4. Check graceful degradation
- When a non-critical dependency fails, does the entire request fail?
- Are there features that could return partial/cached data instead of erroring?
- What happens during deployment? Are there health checks? Graceful shutdown?

## What to Look For

### Swallowed Errors
- Empty `catch` blocks or catch blocks with only `console.log` that don't re-throw or return error state
- `.catch(() => {})` or `.catch(() => null)` on promises — error silently becomes null, downstream code has no idea
- Error callbacks that are never checked: `callback(err)` where the caller ignores the first argument
- `try/catch` around a large block where different errors need different handling but all get the same generic catch
- **Key question**: If this error occurs in production, would anyone know?

### Missing Error Context
- Re-throwing errors without wrapping — original stack trace or context is lost
- Generic error messages: `throw new Error("Something went wrong")` — useless for debugging
- Logging the error message but not the stack trace
- Catching typed errors and converting to a generic string, losing the type information
- Missing correlation: errors logged without request ID, user ID, or relevant business context

### Missing Timeouts
- HTTP clients calling external services with no timeout configured (will hang forever if the service is slow)
- Database queries with no statement timeout
- `Promise.all` or `Promise.race` without an overall timeout wrapper
- Queue consumers that process indefinitely without a per-message timeout
- Lock acquisition with no timeout (deadlock potential)

### Missing or Broken Retry Logic
- External calls that fail once and give up (network blips cause user-facing errors)
- Retry logic without exponential backoff (hammers a struggling service)
- Retry logic without a maximum retry count (infinite retry loop)
- Retry on non-idempotent operations (POST retried = duplicate records)
- Retry without jitter (all instances retry at the same time = thundering herd)

### Partial Failure Handling
- `Promise.all` where one failure should not cancel the others — `Promise.allSettled` more appropriate
- Batch operations that fail entirely when one item has an issue
- Transaction scope too large: one unrelated failure rolls back a large set of valid changes
- Missing compensation logic: step 1 succeeds, step 2 fails, step 1 is not rolled back

### Cascading Failure Patterns
- No circuit breaker on external service calls — a slow dependency makes every request slow
- Missing bulkheads: one misbehaving endpoint exhausts the entire connection/thread pool
- Unbounded queues between components: producer is fast, consumer is slow, memory grows until OOM
- Health check that depends on a non-critical service — if that service is down, the health check fails, load balancer takes the instance out of rotation

### Observability Gaps
- **Missing structured logging**: Important operations produce no log output, or only unstructured strings
- **Missing error tracking**: Errors caught and handled but not reported to any monitoring system
- **No request tracing**: Can't follow a request through multiple services or async steps
- **Silent data issues**: Data validation failures that return empty results instead of errors (the query works, it just returns nothing because the ID format was wrong)
- **Missing business metrics**: No visibility into rates of important business events (signups, payments, exports)

### Startup and Shutdown
- No readiness/health check: traffic arrives before the service is ready (DB connections not established, caches not warmed)
- No graceful shutdown: in-flight requests are dropped when the process receives SIGTERM
- Missing dependency checks at startup: service starts successfully but will fail on first request because a required service is unreachable
- Configuration validation deferred to first use instead of checked at startup

### Type Safety at Boundaries
- External API responses used without validation (assumes the shape matches the type definition)
- Database query results cast to a type without runtime validation
- User input parsed as JSON and cast to an interface without schema validation
- Environment variables read and used without checking if they exist (undefined becomes the string "undefined")

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **critical** | Errors swallowed on critical paths (payment, auth, data mutation) — failures happen silently. Missing timeouts on synchronous external calls in hot paths. |
| **high** | Cascading failure patterns. Missing retries on idempotent operations to essential services. Startup without dependency validation. Empty catch blocks in data processing pipelines. |
| **medium** | Observability gaps on important paths. Missing graceful shutdown. Partial failure handling absent. Error context lost during propagation. |
| **low** | Missing structured logging on non-critical paths. Retry without jitter. Health checks that could be more granular. |

## Reporting

Use these rule ID prefixes:

- `reliability/swallowed-error` — Errors caught but not handled, propagated, or reported
- `reliability/missing-context` — Errors re-thrown or logged without sufficient debugging context
- `reliability/missing-timeout` — External calls, queries, or locks without timeout configuration
- `reliability/missing-retry` — Transient-failure-prone operations without retry logic
- `reliability/broken-retry` — Retry without backoff, jitter, max attempts, or idempotency check
- `reliability/cascading-failure` — Missing circuit breakers, bulkheads, or backpressure
- `reliability/partial-failure` — All-or-nothing handling where partial success is appropriate
- `reliability/observability-gap` — Missing logging, tracing, or monitoring on important paths
- `reliability/unsafe-startup` — Missing health checks, dependency validation, or graceful shutdown
- `reliability/boundary-trust` — External data used without runtime validation

For each rule **`shortDescription`**, write a concise 1-sentence title (e.g. "Missing timeout on external payment API call").

For each rule **`fullDescription`**, explain:
1. What the failure mode is and what triggers it
2. What happens from the user's or operator's perspective (silent corruption? Hung request? Missing data?)
3. How bad it is to diagnose — would you know this happened from logs/monitoring alone?

For each result **`message`**, provide instance-specific context — mention the file, function, or call site involved.
