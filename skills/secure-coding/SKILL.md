---
name: secure-coding
description: >-
  Write secure code by default when building or changing any feature or product.
  Applies secure defaults for authentication, authorization, secrets, input
  validation, injection prevention, data handling, logging, cryptography,
  dependencies, and API/web security during the CODING phase. Use whenever
  developing a new feature or product, modifying existing feature code, adding an
  endpoint, service, background job, integration, or datastore, or implementing
  any change that processes input, moves data, or crosses a trust boundary.
---

# Secure Coding

Apply secure defaults every time you write or change code. This skill is
self-contained: follow the rules here directly, regardless of any other
guidance in the project.

The full ruleset lives next to this file:
[secure-coding-rules.md](secure-coding-rules.md). Read it when a change touches a
domain below and you need the detailed rules.

## When this applies

Trigger on any of: building a new feature or product; changing existing feature
code; adding or modifying an endpoint, handler, background job, integration,
datastore, queue, cache, auth flow, permission, secret, config key, dependency,
file upload/download, or data model — i.e. essentially every change that
processes input, moves data, or crosses a trust boundary.

## Workflow

Copy this checklist and track progress:

```
- [ ] 1. Identify the security domains this change touches
- [ ] 2. Implement with secure defaults (deny by default, least privilege)
- [ ] 3. Add tests for auth, authz, validation, and cross-tenant failure paths
- [ ] 4. Run the pre-commit self-review checklist
```

**1. Identify domains.** Map the change to the domains in
[secure-coding-rules.md](secure-coding-rules.md) (auth, secrets, input, data,
logging, crypto, API/web, dependencies, infra) and load the relevant rules.

**2. Implement securely.** Prefer secure framework patterns over custom security
code. Keep changes minimal and scoped. Choose secure defaults over convenience.

**3. Test the failure paths.** Add negative tests: unauthorized access, forbidden
access, cross-tenant access, and validation rejections — not just the happy path.

**4. Self-review** using the checklist below before finishing.

## Non-negotiable secure defaults

- **Auth:** every non-public path requires authentication; never trust
  client-supplied identity/role/tenant claims without server-side validation.
- **Authz:** enforce server-side, deny by default; check both permission and
  resource/tenant scope. Enforce tenant isolation on every read/write path.
- **Secrets:** never in source, tests, configs, Dockerfiles, CI, docs, or logs.
  Load only from approved secret managers / env injection.
- **Input:** treat all external input as untrusted; validate schema, type,
  length, range, format at boundaries.
- **Injection:** parameterized queries / ORM-safe patterns only. Never build SQL,
  shell commands, or queries via string concatenation with untrusted input.
- **Output/errors:** no stack traces or internal details to clients; safe,
  generic user-facing error messages.
- **Data:** collect the minimum needed; never log or place PII/secrets in logs,
  metrics labels, cache keys, analytics, or URLs.
- **Crypto/transport:** use TLS and platform-approved crypto libraries; never
  disable cert validation or roll your own crypto.
- **Dependencies:** avoid new deps unless necessary; pin versions; never
  curl-pipe scripts into shells or execute arbitrary downloaded code.
- **Fail closed:** if a security-relevant config is missing, prefer failing the
  build/startup over running insecurely.

## Pre-commit self-review checklist

Before finishing, verify:

```
- [ ] auth enforced on all non-public paths
- [ ] authorization enforced server-side, deny by default
- [ ] tenant isolation preserved where relevant
- [ ] no secrets in code, config, tests, or logs
- [ ] no sensitive data logged, or placed in URLs/metrics/cache keys
- [ ] input validated for all new external inputs
- [ ] injection risks avoided (parameterized queries, no shell concat)
- [ ] safe error handling; no internal details leaked to clients
- [ ] new dependencies justified and pinned
- [ ] tests cover success + validation + unauthorized + cross-tenant cases
- [ ] build/test commands pass locally
```

## Explicitly forbidden

Never: commit secrets; disable auth/authz checks; bypass tenant isolation; add
hidden admin access, magic headers, or undocumented override params; expose
internal/debug endpoints publicly; log tokens/passwords/secrets/raw PII;
concatenate untrusted input into SQL or commands; disable TLS or cert
validation; weaken security middleware for convenience; or silently change
retention, data sharing, or public exposure.

## Reference

Full domain-by-domain rules: [secure-coding-rules.md](secure-coding-rules.md).
