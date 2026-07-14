# Secure Coding Rules (Reference)

Domain-by-domain rules for the coding phase. Load the sections relevant to the
change you are implementing. These rules are self-contained — apply them
directly.

---

## Security-first mindset

- Treat all code changes as security-sensitive unless clearly proven otherwise.
- Prefer secure defaults over permissive behavior.
- Minimize blast radius in all changes.
- Follow least privilege for access, credentials, secrets, permissions, and
  service interactions.
- Prefer denial by default and explicit allow rules.
- Never bypass security checks to satisfy tests or speed up development.
- Do not trade security for convenience; if you must, document the decision in
  the change.

---

## Authentication

- All non-public endpoints must require authentication.
- Do not make an endpoint public unless it is explicitly intended to be.
- Do not add hardcoded credentials, fallback passwords, test backdoors, hidden
  admin routes, or undocumented bypasses.
- Validate tokens using well-maintained libraries and configured
  issuers/audiences; enforce expiry and standard validation checks.
- Do not trust client-provided identity, role, tenant, or permission claims
  unless validated by the server-side auth layer.
- For admin or destructive operations, require stronger auth.

---

## Authorization

- Enforce authorization on every protected action, server-side, not only in UI.
- Use deny-by-default behavior.
- Check both role/permission and resource scope where relevant.
- Enforce tenant isolation in every read/write path for multi-tenant systems.
- Never rely on client-supplied tenant identifiers without server-side validation.
- Add negative tests for unauthorized and cross-tenant access.
- Do not expose admin operations to normal users.

---

## Secrets management

- Never store secrets in source code, test fixtures, sample configs, Dockerfiles,
  CI files, or docs.
- Never commit API keys, passwords, tokens, private keys, certificates, or
  connection strings containing credentials.
- Load secrets only from a secret manager or environment injection.
- Do not print secrets, even in debug logs, exceptions, telemetry, or test output.
- Use separate secrets per environment where supported.
- Follow existing secret names/paths; do not invent new secret storage casually.

---

## Data handling and privacy

- Collect, process, and store only the minimum data needed for the feature.
- Do not add new PII, sensitive, regulated, or tenant-sensitive fields unless
  required.
- When introducing sensitive data, define its classification, retention, and
  deletion, and justify collecting it.
- Do not copy sensitive data into logs, metrics labels, cache keys, analytics
  events, or URLs.
- Avoid storing sensitive data in long-lived local files or temp directories.
- Prefer pseudonymized or masked values when full values are unnecessary.
- Do not use production data in tests unless sanitized and approved.

---

## Logging and observability

- Never log secrets, tokens, session identifiers, passwords, private keys, or raw
  sensitive personal data.
- Log only what is needed for debugging, operations, and auditability.
- Use structured logs; include request/trace identifiers where supported.
- Redact or mask sensitive fields before logging.
- Do not log full request/response bodies for sensitive endpoints.
- Do not place PII in metric names, labels, tags, or tracing spans.
- Log security-relevant failures with enough context to investigate, without
  leaking sensitive content.

---

## Input validation and output safety

- Treat all external input as untrusted; validate at system boundaries.
- Enforce schema, type, length, range, format, and enum constraints.
- Reject unexpected fields where practical; use validated DTOs/models.
- Use parameterized queries or ORM-safe patterns only.
- Do not build SQL, shell commands, or queries via string concatenation with
  untrusted input.
- Escape or sanitize output for its output context.
- Do not return stack traces or internal implementation details to clients.
- Use safe, generic error messages for user-facing responses.

---

## API and web security

- Require auth on non-public APIs.
- Enforce CORS explicitly; no wildcard origins unless intentionally public.
- Use CSRF protection where cookie/session-based flows require it.
- Set and preserve required security headers.
- Enforce request size limits, upload restrictions, and content-type validation.
- Apply rate limiting/throttling on abusable endpoints.
- Validate file uploads for type, size, and handling path.
- Do not expose internal admin or debug endpoints publicly.
- Do not trust client-side validation as a security control.

---

## Encryption and transport security

- Use TLS for all network communication carrying sensitive traffic.
- Do not introduce plaintext transport for sensitive traffic.
- Verify certificates through platform defaults; do not disable cert validation.
- Do not add insecure cryptographic algorithms or modes (e.g. MD5/SHA-1 for
  security, DES/RC4, ECB), and do not write custom crypto.
- Use well-maintained encryption libraries and secure settings.
- Use a cryptographically secure RNG for security-sensitive values.

---

## Dependencies and supply chain

- Prefer existing libraries over adding new dependencies.
- Do not add a new dependency unless necessary.
- New dependencies must be actively maintained, license-compatible, and free of
  known critical vulnerabilities.
- Pin or constrain versions per repo conventions; remove unused deps when
  practical.
- Do not download and execute arbitrary code at runtime.
- Do not curl-pipe scripts into shells in application logic or CI.
- Do not work around SAST, SCA, container, or DAST checks.

---

## Infrastructure and deployment safety

- Do not weaken network boundaries, ingress restrictions, or environment
  isolation.
- Do not open new ports, listeners, or public routes unless required.
- Keep admin functions separated from public traffic.
- Do not disable WAF, rate limiting, auth filters, or security middleware to make
  tests pass.

---

## Secure coding conduct

- Prefer existing secure framework patterns over custom security code.
- Keep changes minimal and scoped.
- Do not refactor unrelated areas unless necessary for correctness or safety.
- Do not create hidden flags, magic headers, or undocumented override parameters.
- Do not add debug-only code paths that can survive into production.
- Do not hardcode environment-specific URLs, credentials, or tenant IDs.
- Do not use broad exception swallowing that hides security failures.
- Surface permission and validation failures cleanly and safely.

---

## Testing

- Add or update tests for: successful behavior, validation failures, unauthorized
  access, forbidden access, tenant isolation failures (where relevant), and edge
  cases.
- Do not delete security tests without replacement.
- Do not suppress failing security checks without documenting the reason.
- Fail closed: if a security-relevant configuration is missing, prefer failing
  the build/startup over silently running insecurely.

---

## When unsure

If a requirement is ambiguous:

- prefer the more secure implementation
- avoid expanding access
- avoid storing more data
- avoid exposing more detail
- document assumptions in the change
