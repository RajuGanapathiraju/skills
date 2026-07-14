---
name: secure-architecture
description: >-
  Generate a tier-appropriate secure architecture design document from a feature
  brief, using the team's Secure Architecture Spec. Use when starting a new
  service/feature/major refactor, onboarding a new third-party or AI/LLM
  dependency, re-baselining a service whose risk changed, or when the user
  mentions secure architecture, security design, design review, threat-model
  hand-off, or a security tier (Baseline/Moderate/High).
---

# Secure Architecture

Apply the **Secure Architecture Spec** to a feature/product brief and produce a
project-specific secure architecture document for the **DESIGN phase** — security
principles, posture, and intent, *before* any code is written.

The spec template lives next to this file: [secure-architecture-template-v2.md](secure-architecture-template-v2.md).
Read it in full before generating a document; it is the source of truth for required
sections and inline guidance.

## Scope

- This covers the **architecture/design phase only**. Coding-level controls,
  threat-model mitigations, and test/CI gating live in other guardrails — reference
  them at the *intent* level (Section 12 hand-off), do not specify them here.
- Output is a single Markdown file: `secure-architecture-<project>.md`.

## Workflow

Copy this checklist and track progress:

```
- [ ] 1. Read the brief (Jira ticket / PRD) and the template
- [ ] 2. Select a security tier (Baseline / Moderate / High)
- [ ] 3. Confirm tier if ambiguous — do not guess
- [ ] 4. Fill in EVERY section for the selected tier
- [ ] 5. Replace placeholders with real values or N/A
- [ ] 6. Write secure-architecture-<project>.md
- [ ] 7. Summarize tier, key decisions, and open assumptions
```

**1. Read inputs.** Pull the brief (if a Jira/URL is given, fetch it). Read the
template so every required section and its inline guidance is in context.

**2. Select a tier.** Higher tiers inherit all lower-tier requirements.
- **Baseline** — internal tool, public data only, no PII, intranet-only, low impact.
  Still requires foundational controls (authn, authz, logging, dependency hygiene).
- **Moderate** — private/internal data, internet-facing APIs, role-based access,
  medium impact. Any service where user-controlled input reaches an AI/LLM model is
  automatically at least Moderate.
- **High** — PII/regulated data, critical business system, compliance scope, high
  impact. AI systems that act on outputs autonomously (agents, auto-remediation) are
  automatically High unless a compelling case is made otherwise.

**3. Confirm if ambiguous.** If the brief doesn't give enough to set the tier
confidently, STOP and ask the human/security owner. Tier selection drives everything
else. Default toward the higher tier when data sensitivity or exposure is unclear.

**4. Fill in every section.** Complete all 12 sections plus the glossary as defined in
the template. Tailor content to the actual feature — concrete assets, data flows,
trust boundaries, and roles, not restated placeholders.

**5. Handle N/A honestly.** Mark genuinely irrelevant items `N/A` with a one-line
reason (e.g., "No AI/LLM dependency" for Section 9). Do not silently skip.

**6. Write the document** as `secure-architecture-<project>.md` in the repo (or where
the user indicates). Do not overwrite the template itself.

**7. Summarize** the chosen tier and rationale, the security-relevant design
decisions, and any open assumptions/risks the owner should confirm.

## Quality bar

- Capture *intent and posture*, not implementation details (no specific header
  values, library names, TTLs-as-config, or CI tool wiring).
- Trust boundaries and tenant/data isolation are described where trust actually
  changes — be specific to this system.
- Treat any external/AI output crossing back into the app as untrusted.
- Keep the document **evergreen**: note that any change in data, users, exposure, or
  third-party/AI integration should trigger a tier re-review.

## Reference

Full section list, inline guidance, examples, and glossary:
[secure-architecture-template-v2.md](secure-architecture-template-v2.md).
