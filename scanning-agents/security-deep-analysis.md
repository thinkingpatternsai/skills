---
name: Security Deep Analysis
description: >
  Contextual security analysis that goes beyond pattern matching. Traces auth flows,
  data propagation, trust boundaries, and business logic to find vulnerabilities
  that traditional scanners miss.
category: security
---

# Security Deep Analysis

Perform a deep, contextual security analysis of the codebase. Traditional scanners catch
known patterns (eval, SQL concatenation, known CVEs). This analysis focuses on what they
miss: broken auth flows, data exposure through business logic, trust boundary violations,
and subtle interaction bugs that only emerge from understanding how the code actually works.

## Analysis Strategy

### 1. Map the attack surface
- Find all entry points: HTTP routes/handlers, API endpoints, WebSocket handlers, CLI commands, queue consumers, cron jobs
- For each entry point, identify: what auth is required, what input is accepted, what data is accessed
- Use `glob` and `grep` to find route definitions, controller registrations, middleware chains

### 2. Trace auth and authorization flows
- Start from auth middleware/guards. Trace which endpoints they protect
- Look for endpoints that SKIP auth — are there routes registered after middleware, unprotected API versions, debug endpoints?
- Check authorization: does the code verify the user has access to the SPECIFIC resource, not just that they're authenticated?
- Look for: object-level auth (IDOR), function-level auth (admin endpoints accessible to regular users), field-level auth (users modifying fields they shouldn't)

### 3. Follow sensitive data flows
- Identify where secrets, PII, tokens, and credentials enter the system
- Trace where they go: logged? cached? serialized? included in error messages? passed to third parties?
- Check: are API responses filtered? Does the ORM select * when it should select specific fields?
- Look at error handlers — do they leak stack traces, internal paths, database details?

### 4. Examine trust boundaries
- Where does the code transition between trusted and untrusted contexts? (user input, external API responses, database reads from shared tables, file uploads)
- Is input validated at the boundary, or deep inside business logic where it might be skipped?
- Are external API responses treated as trusted? (SSRF, injection through upstream data)

## What to Look For

### Authentication Gaps
- Routes or endpoints with no auth middleware applied
- Auth checks that verify identity but not authorization for the specific action/resource
- Token validation that doesn't check expiry, audience, or issuer
- Session fixation: sessions not regenerated after login
- Password reset flows that don't invalidate old tokens
- API keys with excessive scope or no rotation mechanism

### Authorization & Access Control
- **IDOR**: User-supplied IDs used to fetch resources without ownership checks (e.g., `GET /api/orders/:id` that doesn't verify the order belongs to the requesting user)
- **Privilege escalation**: role checks that use string comparison instead of proper role hierarchy, or role checks missing entirely on admin operations
- **Horizontal access**: multi-tenant data accessed without org/tenant scoping
- **Mass assignment**: request body directly passed to ORM update/create without allowlisting fields

### Data Exposure
- API responses that include internal IDs, timestamps, or metadata the client doesn't need
- Error handlers that return stack traces, SQL queries, or internal paths
- Logs that include tokens, passwords, PII, or request bodies
- Caching layers that store sensitive data without appropriate TTL or access controls
- Debug/development endpoints or flags still reachable in production configuration

### Business Logic Flaws
- Race conditions in financial operations (double-spend, time-of-check-to-time-of-use)
- State machine violations (order can go from "cancelled" back to "processing")
- Numeric overflow/underflow in calculations (especially pricing, quantities, balances)
- Inconsistent validation between client and server, or between different server endpoints that modify the same data

### Injection (Beyond Pattern Matching)
- Dynamic query construction using user input, even if parameterized — look for cases where column names, table names, or ORDER BY directions come from user input
- Template rendering with user-controlled template strings (not just variables)
- URL construction from user input used in server-side fetches (SSRF)
- File path construction from user input (path traversal), especially when combined with archive extraction

### Cryptographic Issues
- Symmetric encryption with hardcoded or derived-from-predictable-source keys
- JWT signed with weak secrets or using `alg: none`
- Comparison of hashes/tokens using `==` instead of constant-time comparison
- Random values generated with Math.random() or similar non-cryptographic sources for security-sensitive purposes

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **critical** | Unauthenticated access to sensitive data or operations. RCE. Auth bypass. |
| **high** | Authenticated user can access other users' data (IDOR). Privilege escalation. Sensitive data exposure in API responses. |
| **medium** | Information disclosure that aids further attacks. Missing rate limiting on sensitive endpoints. Weak cryptographic choices. |
| **low** | Defense-in-depth gaps. Verbose error messages in non-production configs. Missing security headers. |
| **info** | Observations that aren't vulnerabilities but indicate security debt or areas to monitor. |

## Reporting

Use these rule ID prefixes for findings:

- `security/auth-bypass` — Missing or ineffective authentication
- `security/broken-access-control` — Authorization failures, IDOR, privilege escalation
- `security/data-exposure` — Sensitive data leaked in responses, logs, errors
- `security/injection` — SQL, command, template, SSRF, path traversal
- `security/business-logic` — Race conditions, state violations, numeric errors
- `security/crypto-weakness` — Weak algorithms, hardcoded keys, bad randomness
- `security/mass-assignment` — Unfiltered input passed to data modification operations

For each rule **`shortDescription`**, write a concise 1-sentence title (e.g. "Hardcoded database credentials in config module").

For each rule **`fullDescription`**, explain:
1. What the vulnerability is
2. Where the data flow or control flow creates the risk (not just "line 42 is bad")
3. What an attacker could do (concrete impact)

For each result **`message`**, provide instance-specific context — mention the file, function, or variable involved.
