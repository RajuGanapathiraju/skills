---
name: secure-code-review
description: Performs deep, adversarial security review of codebases. Use when the user asks for a security audit, vulnerability scan, secure code review, penetration test of source code, or wants to find security bugs, exploits, or attack paths in a repository.
---

# Secure Code Review

Perform a principal-level adversarial security review. You are not auditing code — you are trying to break the system.

## Phase 1: Reconnaissance

Map the codebase before analyzing anything:

1. **Entry points** — controllers, API routes, handlers, main functions, WebSocket endpoints, GraphQL resolvers
2. **Auth layer** — filters, middleware, JWT/session handling, RBAC, OAuth config, API key validation
3. **Config & secrets** — `application.yml`, `.env`, `config.env`, K8s manifests, IAM policies, Dockerfiles
4. **External integrations** — HTTP clients, gRPC, message queues, LLM/AI calls, webhooks, email/SMS
5. **Data stores** — DB repositories, Redis, S3, caches, file system
6. **Async flows** — Temporal workflows, cron jobs, retry logic, background tasks, event-driven handlers
7. **File handling** — upload endpoints, download handlers, temp file creation, archive extraction

Use parallel subagent exploration tasks to cover these areas simultaneously. Spend the majority of your effort here — you cannot find what you haven't mapped.

While mapping, keep two running inventories that feed the report:
- **Endpoint inventory:** every externally reachable entry point and, for each, whether an auth/authz check sits in front of it. Flag any that accept requests with no auth header/token/cookie/session (feeds output section 3).
- **Coverage log:** track which files/directories you actually reviewed vs. skipped, so you can accurately report what was *not* scanned (feeds output section 8). Note the reason for anything skipped.

### Search patterns

Grep across the codebase for:

```
auth, token, session, jwt, bearer, apikey, secret, password, credential
admin, role, permission, authorize, @PreAuthorize, @Secured, @RolesAllowed
upload, file, path, File, Paths.get, ClassPathResource, MultipartFile, ZipInputStream
exec, spawn, eval, Runtime, ProcessBuilder, ScriptEngine, reflection, Class.forName
query, sql, filter, @Query, JdbcTemplate, nativeQuery, createNativeQuery, HQL
webhook, callback, redirect, url, URI.create, WebClient, RestTemplate, HttpClient
serialize, deserialize, ObjectInputStream, readObject, @JsonTypeInfo, XMLDecoder
encrypt, hash, MD5, SHA1, Random, SecureRandom, DES, RC4, Cipher
cors, csrf, origin, X-Forwarded, Host, Access-Control
race, synchronized, lock, atomic, Thread, CompletableFuture, @Async
```

## Phase 2: Analysis

For each entry point found, trace the full data flow:

```
User Input → Validation → Processing → Sink (DB/API/File/Response)
```

At each stage ask:
- Who can reach this? (auth requirement)
- What inputs are trusted vs untrusted?
- Is ownership verified? (tenant/user isolation)
- What happens on the error path?
- Can this be replayed, raced, or abused?

### Vulnerability categories to cover

#### Access Control & Auth
- IDOR / BOLA / BFLA — resource accessed by ID without ownership check
- Privilege escalation — horizontal (user→other user) and vertical (user→admin)
- Multi-tenant isolation — can tenant A access tenant B's data?
- Auth bypass — missing filters, blank header acceptance, commented-out security
- RBAC gaps — no role checks, same permissions for all users
- Mass assignment / parameter pollution — binding untrusted input to internal fields
- JWT flaws — `alg:none`, key confusion (HMAC/RSA), `kid` injection, missing expiry validation, weak signing keys
- OAuth/SSO flaws — `redirect_uri` manipulation, missing `state` parameter, PKCE bypass, token leakage in logs/referrers
- Session fixation / weak invalidation — session ID not rotated after login, no server-side invalidation, long-lived tokens
- MFA bypass — fallback to weaker auth, missing MFA on sensitive operations

#### Injection
- SQL / NoSQL / ORM injection — string concatenation in queries, unparameterized HQL/JPQL
- Command injection / RCE — `Runtime.exec()`, `ProcessBuilder`, user input in shell commands
- Template / expression language injection — SpEL, Thymeleaf, FreeMarker, Pebble, Jinja2 (SSTI → RCE)
- LDAP / XPath injection
- LLM prompt injection — user input concatenated into prompts, LLM output trusted as code/filters/queries
- Header / CRLF / log injection — newlines in headers, unsanitized log output
- XSS — stored, reflected, DOM-based; insufficient output encoding; CSP bypass
- XXE — XML parsers with external entities enabled (`DocumentBuilderFactory`, `SAXParser`, `XMLInputFactory`)

#### Deserialization → RCE
- Java native deserialization — `ObjectInputStream.readObject()` on untrusted data
- Polymorphic JSON deserialization — Jackson `@JsonTypeInfo` with `defaultTyping`, `enableDefaultTyping()`
- XML deserialization — `XMLDecoder`, `XStream` without allowlisting
- YAML deserialization — `SnakeYAML` `load()` on untrusted input

#### Cryptographic Failures
- Hardcoded secrets in source or config
- Weak algorithms (MD5, SHA-1, DES, RC4, ECB mode)
- InsecureTrustManagerFactory / disabled cert verification
- Missing TLS for service-to-service communication
- `java.util.Random` for security-sensitive operations (use `SecureRandom`)
- Weak/missing key derivation (plaintext passwords, unsalted hashes)
- Timing-safe comparison not used for secrets/tokens (`==` instead of `MessageDigest.isEqual`)

#### Security Misconfiguration
- Wildcard CORS (`allowedOrigin("*")`)
- CSRF disabled without justification
- Actuator/health endpoints exposed without auth
- Debug mode / verbose error responses in production
- Overprivileged K8s RBAC, IAM policies, container security contexts
- Missing security headers (CSP, HSTS, X-Content-Type-Options, X-Frame-Options)
- Default credentials in any environment
- Directory listing enabled

#### SSRF
- HTTP clients calling user/config-controlled URLs without allowlists
- Component registry or service discovery poisoning
- DNS rebinding attacks
- Cloud metadata endpoint access (169.254.169.254, fd00:ec2::254)

#### Race Conditions & Concurrency
- TOCTOU (time-of-check-to-time-of-use) — check permission then act without holding lock
- Double-spend / double-execution — concurrent requests bypass uniqueness constraints
- Account takeover via parallel password reset flows
- Inventory/quota bypass via concurrent requests
- Database-level race conditions — `SELECT` then `INSERT/UPDATE` without row locking or `SELECT FOR UPDATE`
- File system races — check file, then read/write without atomic operations
- Distributed race conditions — multiple service instances without distributed locking

#### File & Path Handling
- Path traversal — `../` in filenames, unsanitized user input in file paths
- Zip Slip — archive extraction without entry name validation
- Unrestricted file upload — no type validation, executable content, polyglot files
- Content-type confusion — trusting client-provided Content-Type
- Symlink attacks — following symlinks during file operations
- Unsafe temp files — predictable names, world-readable permissions
- Large file DoS — no size limits on uploads

#### Open Redirect & URL Handling
- Unvalidated redirect URLs from user input
- URL parser differentials (what the validator sees vs what the browser follows)
- JavaScript: URIs / data: URIs in redirect targets

#### HTTP-Level Attacks
- Request smuggling — CL/TE or TE/CL desync between proxy and backend
- Host header attacks — password reset poisoning, cache poisoning via Host header
- Cache poisoning — unkeyed headers influencing cached responses
- Clickjacking — missing X-Frame-Options / CSP frame-ancestors
- HTTP parameter pollution

#### WebSocket Security
- Missing origin validation on WebSocket upgrade
- No authentication/authorization after handshake
- Message injection / cross-site WebSocket hijacking

#### ReDoS & Resource Exhaustion
- Catastrophic backtracking in user-facing regex patterns
- Unbounded collection/stream operations on user-controlled input sizes
- Hash collision attacks (HashDoS)
- XML bomb / billion laughs (for XML parsers)
- Large payload / body size limits not enforced

#### Unsafe Reflection & Dynamic Code
- `Class.forName()` / `Method.invoke()` with user-controlled class/method names
- Dynamic proxy creation with untrusted input
- Expression evaluation engines (SpEL, OGNL, MVEL) with user input
- Script engine execution (`javax.script.ScriptEngine`) with user input

#### Business Logic (often the highest-impact findings)
- Workflow bypass — skipping approval, review, or validation steps
- Replay / double-execution — idempotency failures on mutations
- Rate limit absence — unbounded resource creation or API calls
- Tenant/resource enumeration — predictable IDs, unscoped listing endpoints
- Economic abuse — unbounded cost generation (LLM calls, cloud resources, email/SMS)
- Price/quantity manipulation — client-controlled pricing, negative quantities
- Coupon/credit/referral abuse — stacking, replay, self-referral
- State machine violations — forcing transitions, skipping steps
- Integer overflow / type confusion in business calculations

#### Supply Chain
- Dependency confusion — public repo resolves before private
- Floating versions (`5.+`, `latest` tags)
- Known CVEs in pinned versions
- Malicious scripts (postinstall, build hooks)
- Compromised base images (Docker `latest` tags, unverified registries)

#### Data Exposure
- Secrets in Git (API keys, passwords, tokens in tracked files)
- Exception messages returned to clients (`ex.getMessage()` in responses)
- Sensitive data logged (request bodies, tokens, PII)
- Health/metrics endpoints leaking infrastructure details
- Stack traces in production responses
- Verbose GraphQL introspection enabled in production
- Sensitive data in URL query parameters (logged by proxies/browsers)

## Phase 3: Attack Chain Construction

After individual findings, identify multi-step exploit paths:

```
Example: IDOR (list all resources) → Extract victim's ID →
         Use victim's ID in mutation endpoint → Modify victim's data

Example: Config poisoning → SSRF → Internal service access →
         Credential theft → Lateral movement

Example: Prompt injection → LLM generates malicious filter →
         Filter executed against DB → Data exfiltration

Example: File upload (polyglot) → Stored XSS / RCE →
         Admin session hijack → Full system compromise

Example: Race condition on token refresh → Session fixation →
         Account takeover

Example: Deserialization of untrusted data → RCE →
         Reverse shell → Cloud credential theft
```

## Phase 4: Output

Structure the report strictly as follows:

### 1. Executive Summary
- 3-5 sentence overview of security posture
- Top 3 critical attack paths
- Key systemic issues

### 2. Findings (sorted by severity: CRITICAL → HIGH → MEDIUM → LOW)

For each finding:

```
### [SEVERITY] Title

- **Category:** (e.g., Broken Access Control, Injection, Security Misconfiguration)
- **Location:** `file:line` + function name
- **Description:** What the vulnerability is
- **Why vulnerable:** Why the current code is exploitable
- **Externally Exploitable:** Yes / No / Conditional — can an unauthenticated remote attacker (i.e., someone on the public internet with no valid credentials) reach and trigger this?
  - **Exposure:** Public internet / Authenticated users only / Internal network only / Local only
  - **Reachability:** Which specific route/entry point exposes it, and what network position + privileges are required to reach the sink (e.g., "public POST /api/import, no auth header needed")
  - **Preconditions:** Any conditions that must hold for the external path to work (feature flags, specific config, a valid but low-privilege account, etc.). Mark "None" if directly exploitable.
- **Exploit scenario:** Step-by-step attack narrative
- **Impact:** What an attacker gains
- **Evidence:**
  (code reference block from the file)
- **Fix:** Specific, actionable remediation
- **Confidence:** Confirmed / High / Medium / Low
```

> Always determine **Externally Exploitable** by tracing the entry point back to the network boundary: is the route registered on a publicly-exposed listener, and does every layer in front of the sink (gateway, middleware, filter, auth guard) actually enforce authentication/authorization? A vulnerability that requires no valid credentials from the public internet is strictly higher priority than one requiring an authenticated session — reflect this in severity ordering when two findings are otherwise comparable.

### 3. Unauthenticated / Open Endpoints

Enumerate **every** endpoint (HTTP route, GraphQL operation, WebSocket handler, gRPC method, message/queue consumer, webhook, etc.) that can be reached **without any valid authentication or authorization** — i.e., an end user can invoke it without supplying any auth header, cookie, token, API key, or session.

For each, determine whether the exposure is intentional (e.g., login, health check, public docs) or an accidental gap (missing middleware, commented-out guard, route registered before the auth filter, permit-all rule that is too broad).

Present as a table:

| Endpoint (method + path) | Handler (`file:line`) | Auth Check Present? | Intended to be Public? | Sensitive Action / Data Exposed | Risk |
|---|---|---|---|---|---|
| e.g. `POST /api/v1/users/import` | `ImportController.import` (`src/.../ImportController.java:42`) | None | No — gap | Bulk create users, DB write | HIGH |

Then, below the table:
- **Confirmed auth gaps:** endpoints that should require auth but do not (these should also appear as findings in section 2 with `Externally Exploitable: Yes`).
- **Intentionally public but risky:** public-by-design endpoints that still expose sensitive data, mutate state, or lack rate limiting / input validation.
- **How auth is (or isn't) enforced:** briefly describe the auth mechanism (global filter, per-route decorator, gateway-level) and any patterns that make gaps likely (allowlist vs. denylist, default-permit routing, ordering issues).

### 4. Attack Chains
Multi-step exploits combining individual findings.

### 5. Business Logic Abuse Cases
Separate section for workflow bypass, replay, quota abuse, tenant breakout, fraud.

### 6. Design Weaknesses
Architecture-level issues (e.g., "no defense in depth", "trust model relies on single gateway").

### 7. Remediation Roadmap
- **Immediate** (this week)
- **Short-term** (1-2 sprints)
- **Long-term** (quarter)

### 8. Coverage & Files Not Scanned

Be explicit and honest about what was and was not reviewed, so the reader knows where residual risk may hide.

- **Scanned:** high-level summary of the directories/modules/entry points that were analyzed.
- **Not scanned / partially scanned:** list files, directories, or components that were **not** reviewed for vulnerabilities, each with a reason. Use a table:

| Path | Type | Reason not (fully) scanned |
|---|---|---|
| e.g. `vendor/`, `node_modules/` | Third-party deps | Out of scope for source review (see Supply Chain) |
| e.g. `src/legacy/report/*.jsp` | View templates | Not reached in time / low priority — needs follow-up |
| e.g. `*.min.js`, generated code | Build artifacts | Minified/generated, not source of truth |
| e.g. binary assets, images | Binary | Not source code |

- **Blind spots:** areas where source alone is insufficient to conclude safety (runtime config, infra/IaC not in repo, external services, feature-flagged code paths, dynamically loaded plugins).
- **Recommended follow-up:** what should be reviewed next to close the coverage gaps.

### 9. Final Lists
- Top 10 critical vulnerabilities
- Top 10 externally exploitable (unauthenticated) issues
- Top 10 business logic risks
- Unknowns / assumptions / areas not analyzed

## Rules

- Do NOT stop at obvious issues — dig deeper after each finding
- Do NOT report only lint/static-analysis issues — focus on real exploitability
- Follow full data flows across files and modules
- Clearly mark uncertain findings with confidence levels
- Highlight **missing protections**, not just bugs
- Think as: malicious user, competitor, fraudster, insider attacker
- Assume you control all inputs: headers, params, body, files, timing
- Look for chaining opportunities across components
- For each access-control finding, verify both the happy path AND the bypass path
- For every finding, explicitly determine whether it is **externally exploitable** by an unauthenticated remote attacker, and trace the entry point to the network boundary to justify it
- Enumerate **all** endpoints reachable with no authentication/authorization; distinguish intentional public endpoints from accidental auth gaps
- Be transparent about coverage — always report which files/areas were NOT scanned and why; never imply full coverage you didn't achieve
