---
name: Scalability Review
description: >
  Analyzes code for performance bottlenecks and scalability issues that emerge under load.
  Traces data access patterns, resource management, concurrency handling, and growth-sensitive
  code paths that will break before the business does.
category: scalability
---

# Scalability Review

Analyze the codebase for patterns that work fine at low scale but break under load. This is
not micro-optimization — it's finding the structural issues that cause outages, timeouts,
and cascading failures when traffic grows 10x or data grows 100x.

## Analysis Strategy

### 1. Map data access patterns
- Find all database queries: ORM calls, raw SQL, cache reads/writes
- For each query, determine: is it in a loop? Is it paginated? Does it have appropriate indexes?
- Trace the lifecycle of database connections: are they pooled? Released promptly? What happens on error?
- Use `grep` to find ORM model definitions, query builders, raw SQL strings

### 2. Identify hot paths
- Find request handlers, event processors, queue consumers — the code that runs on every request
- In each hot path, look for: synchronous blocking, unbounded iterations, missing caching, heavy computation
- Check: are expensive operations (API calls, file I/O, crypto) happening in the request lifecycle?

### 3. Examine resource management
- Connection pools: databases, Redis, HTTP clients, message queues — are there limits? What happens at capacity?
- Memory: are there caches, buffers, or collections that grow without bounds?
- File handles, sockets, child processes: are they properly cleaned up?

### 4. Analyze concurrency patterns
- Where does concurrent access occur? Shared state, database rows, external resources
- Are there race conditions that produce incorrect results under concurrent load?
- What happens when two requests modify the same resource simultaneously?

## What to Look For

### N+1 Queries and Query Amplification
- Loop that fetches related records one at a time instead of batch loading (classic N+1)
- API endpoint that makes a database call per item in a list received from the client
- GraphQL resolvers that trigger individual queries for each field/relationship
- Recursive queries where depth is controlled by user data
- **How to find**: Look for database/ORM calls inside `for`, `forEach`, `map`, `.then` chains, or `Promise.all` wrapping individual fetches

### Unbounded Operations
- Queries without LIMIT that return entire tables — especially if filtered only by org/user
- `SELECT *` on tables that may have large text/blob columns
- API endpoints that return all results without pagination
- In-memory sorting or filtering of large datasets that should be done in the database
- File reads that load entire files into memory without streaming
- Regex applied to user-controlled input without length limits (ReDoS)

### Missing or Broken Pagination
- List endpoints that accept `limit`/`offset` but don't enforce a maximum limit
- Offset-based pagination on large tables (performance degrades linearly with page number)
- Pagination that doesn't use stable cursors, producing inconsistent results during writes
- Count queries (`SELECT COUNT(*)`) on large tables for every paginated request

### Connection and Resource Management
- Database connections opened but not returned to pool on error paths (connection leak)
- HTTP clients created per-request instead of reused
- Connection pools with no max size, or max size set too high for the database
- Missing connection timeouts — a slow downstream service exhausts the pool
- File handles or streams opened without `finally`/`using`/`defer` cleanup
- Child processes spawned without limits or cleanup

### Blocking and Synchronous Bottlenecks
- Synchronous file I/O, DNS lookups, or crypto operations in async request handlers
- `await` in a loop where operations could be parallelized with `Promise.all` or equivalent
- Locks held during I/O operations (lock, then make HTTP call, then unlock)
- Single-threaded event loop doing CPU-intensive work (JSON parsing large payloads, image processing, compression)

### Caching Issues
- Expensive computation or query repeated on every request with no caching
- Cache with no TTL or eviction — grows unbounded
- Cache stampede: when a popular cache key expires, all concurrent requests hit the backend simultaneously
- Cache invalidation that's inconsistent with writes (stale reads)
- Caching user-specific data with a shared key (data leaks between users)

### Concurrency and Race Conditions
- Read-modify-write without transactions or optimistic locking (lost updates)
- Counter increments without atomic operations
- Distributed systems assuming single-instance execution (e.g., scheduled job runs on all replicas)
- Queue consumers that can process the same message twice without idempotency
- File or resource access without locking in multi-process/multi-thread environments

### Growth-Sensitive Patterns
- O(n^2) or worse algorithms where n is data-dependent (nested loops over collections that grow)
- Fan-out operations where the fan-out factor grows with data (send notification to all followers)
- Startup initialization that scans entire tables or file systems
- Background jobs that process "all unprocessed records" without batching

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **critical** | Will cause outage or data corruption under moderate load increase. Connection leaks. Unbounded queries on large tables in hot paths. |
| **high** | Significant performance degradation that worsens over time. N+1 in frequently-hit endpoints. Missing pagination on growing datasets. Race conditions producing incorrect data. |
| **medium** | Performance issues that matter at scale but aren't immediately dangerous. Missing caching on expensive operations. Suboptimal query patterns. Blocking I/O in async context. |
| **low** | Inefficiencies that should be addressed but won't cause incidents. Non-reused HTTP clients. Oversized SELECT columns. Offset pagination on moderate tables. |

## Reporting

Use these rule ID prefixes:

- `scalability/n-plus-one` — Query amplification, loop-based data fetching
- `scalability/unbounded-query` — Missing LIMIT, SELECT *, full-table scans
- `scalability/missing-pagination` — List endpoints without proper pagination
- `scalability/resource-leak` — Connection, handle, or memory leaks
- `scalability/blocking-io` — Synchronous operations in async/hot paths
- `scalability/missing-cache` — Repeated expensive operations without caching
- `scalability/race-condition` — Concurrent access without proper synchronization
- `scalability/growth-sensitive` — Algorithms or patterns with superlinear scaling cost

For each rule **`shortDescription`**, write a concise 1-sentence title (e.g. "N+1 query in order listing endpoint").

For each rule **`fullDescription`**, explain:
1. What the pattern is and where it occurs
2. What happens as load/data grows (concrete scenario, not theoretical)
3. The scale at which it becomes a problem if estimable

For each result **`message`**, provide instance-specific context — mention the file, function, or query involved.
