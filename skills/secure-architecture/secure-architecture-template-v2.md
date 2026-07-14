# Secure Architecture — [Project Name]

<!-- PURPOSE: High-level security architecture template for the **DESIGN phase** of
     the SDLC. It captures security *principles, posture, and intent* — not
     implementation details. The goal is to make the application's risk tier and
     security expectations explicit *before* any code is written.

     SCOPE BOUNDARIES (read me first):
     - This file = ARCHITECTURE / DESIGN phase only.
     - Coding-level best practices (parameterized queries, header values, framework
       config, etc.) live in stack-specific guardrail files used during the
       CODE-GENERATION phase (e.g., `security-coding-<stack>.md`).
     - Granular control selection, abuse-case generation, and per-asset mitigations
       are produced by the THREAT-MODELING step, which consumes this document as
       input.
     - Test-phase requirements (SAST/DAST/pentest gates) are referenced here at the
       *intent* level only; the actual tool/CI configuration lives in the
       TEST-phase guardrail file.
     - AI/LLM-specific controls (prompt injection resilience, hallucination control,
       output governance, rate limiting, and abuse monitoring) are captured in
       Section 9. Section 8 records *which* AI providers are used and *what* data
       they receive; Section 9 records *how* those integrations must be secured.

     INSTRUCTIONS FOR THE DEVELOPER OR AGENT:
     - Copy this template into your repo as `secure-architecture-<project>.md`.
    - Read the Jira ticket / product brief and select a security tier
      (Baseline / Moderate / High) using the guide below.
     - If the input is ambiguous, STOP and ask the human owner for the tier — do
       not guess. Tier selection drives everything else.
     - Fill in every section for your selected tier. Higher tiers inherit all
       lower-tier requirements.
     - Replace `[e.g., ...]` placeholders with actual values or `N/A`.
     - Keep this document evergreen: any change in data, users, exposure, or
       third-party / AI integration should trigger a re-review of the tier.

     TIER SELECTION GUIDE:
     - Baseline: Internal tool, public data only, no PII, intranet-only, low business impact.
                Baseline still requires the foundational controls in this template
                (authn, authz, logging, dependency hygiene, etc.) — it is *not* "no security".
     - Moderate: Private/internal data, internet-facing APIs, role-based access, medium impact.
                 Any service where user-controlled input reaches an AI/LLM model is
                 automatically at least Moderate.
     - High:     PII / regulated data, critical business system, compliance scope, high impact.
                 AI systems that produce outputs acted on autonomously (agents, auto-remediation)
                 are automatically High unless a compelling case is made otherwise.

     WHEN TO USE THIS TEMPLATE:
     - Kicking off a new service, feature, or major refactor.
     - Onboarding a new third-party or AI/LLM dependency.
     - Re-baselining an existing service whose risk has changed.
-->

This document defines the **high-level security architecture** for **[Project Name]**.
It establishes the project's risk tier, CIA posture, trust boundaries, AI/LLM security
controls, and the security principles the design must uphold. It is the source of truth
that downstream phases — threat modeling, coding guardrails, and security testing —
build upon.

---

## 1. Project Context & Risk Tier

| Aspect | Value |
|---|---|
| **Project / Service** | [e.g., Login Flow Service / Payment Gateway / AI Assistant / Data Pipeline] |
| **Jira / Brief Link** | [URL] |
| **Problem Statement** | [1–2 sentence summary of what this service does and for whom] |
| **Intended Users** | [e.g., Internal engineers / All customers / Admin users only] |
| **Security Tier** | [Baseline / Moderate / High] |
| **Tier Rationale** | [1–2 sentences justifying the tier based on data, exposure, AI use, and impact] |
| **Business Impact of Compromise** | [Low / Medium / High] |
| **Compliance Scope** | [e.g., SOC 2 / GDPR / HIPAA / PCI-DSS / None] |
| **AI / LLM Integration?** | [Yes — see Section 9 / No] |
| **Security Owner** | [@team-or-person — required for Moderate and High tiers] |
| **Last Review Date** | [YYYY-MM-DD] |

> If the Jira ticket does not give enough information to confidently set the tier,
> pause and request clarification from the security owner before continuing.

---

## 2. CIA Triad Posture

State the protection target for each pillar. This drives every later design choice.

| Pillar | Target (Low / Medium / High) | What must be protected, and why |
|---|---|---|
| **Confidentiality** | [e.g., High] | [e.g., User PII and auth tokens must never leak across tenants or to logs] |
| **Integrity** | [e.g., High] | [e.g., Financial ledger entries must be tamper-evident and append-only] |
| **Availability** | [e.g., Medium] | [e.g., 99.9% uptime SLO; degraded read-only mode acceptable during incidents] |

<!-- High tier only:
**Worst-case scenario:** [One sentence describing the worst plausible outcome if all
three pillars failed simultaneously — e.g., "Mass PII exfiltration with public
disclosure and regulatory fine, compounded by AI-generated misinformation acted on
by downstream systems."]
-->

---

## 3. Data Classification (High-Level)

Describe **what kinds of data** flow through the system, including any data passed to
or received from AI/LLM providers. Field-level handling and retention enforcement
belong to the threat-modeling and coding phases — here, only the classification
posture is captured.

| Data Category | Classification | Source | Crosses a Trust Boundary? |
|---|---|---|---|
| [e.g., User profile] | [Public / Internal / Sensitive / Regulated] | [e.g., User input via UI] | [Yes / No] |
| [e.g., Auth tokens] | [Sensitive] | [e.g., IdP] | [Yes] |
| [e.g., Aggregated analytics] | [Internal] | [e.g., Internal ETL] | [No] |
| [e.g., Prompt context / RAG chunks] | [Internal / Sensitive] | [e.g., User query + retrieved docs] | [Yes — sent to LLM provider] |
| [e.g., Model responses] | [Internal] | [e.g., LLM API] | [Yes — received from LLM provider] |

| Aspect | Value |
|---|---|
| **Highest Classification Handled** | [Public / Internal / Sensitive / Regulated] |
| **Tenancy Model** | [Single-tenant / Multi-tenant — row-level / Multi-tenant — schema-level / N/A] |
| **Data Residency Constraints** | [e.g., EU data must remain in EU region / None] |
| **Retention Intent** | [e.g., Operational data 90 days, regulated records 7 years] |
| **Deletion Intent** | [e.g., User-initiated deletion supported within 30 days / N/A] |

---

## 4. Trust Boundaries & High-Value Assets

A trust boundary is any point where the level of trust changes (user → app, app →
third party, tenant A → tenant B, public network → private network, AI model → app,
etc.). Identifying them here is what enables effective threat modeling later.

**High-value assets:**

| Asset | Classification | Where it Lives | Why it Matters |
|---|---|---|---|
| [e.g., User credentials] | Sensitive | [e.g., IdP only — never in our DB] | [e.g., Account takeover risk] |
| [e.g., Signing keys] | Critical | [e.g., Managed KMS] | [e.g., Forgery of all signed artifacts] |
| [e.g., Customer PII] | Regulated | [e.g., Primary DB] | [e.g., Regulatory + reputational impact] |
| [e.g., System prompt / agent instructions] | Sensitive | [e.g., Config store — never exposed to users] | [e.g., Disclosure enables targeted injection attacks] |
| [e.g., RAG knowledge base / embeddings] | Internal | [e.g., Vector store] | [e.g., Proprietary data; poisoning risk] |

**Trust boundary diagram (high level):**

```
[Sketch the boundaries your data and requests cross, e.g.:]

  Untrusted (Internet / End Users)
        │
        ▼   ── boundary 1: edge (TLS, WAF, DDoS protection) ──
  Application plane (your service)
        │
        ├───▶  ── boundary 2: data plane ──
        │     Data stores (DB, cache, object store, vector store)
        │
        ├───▶  ── boundary 3: third-party egress ──
        │     External providers (IdP, SaaS, payment, messaging)
        │
        └───▶  ── boundary 4: AI / LLM plane ──
              AI / LLM providers (model APIs, embeddings, agents)
              [Outputs cross back into the application plane and
               must be treated as untrusted until validated]
```

> Keep this diagram conceptual. Implementation-level networking details belong in
> Section 6 and the threat-modeling output, not here.

---

## 5. Identity & Access Model

Capture *intent*, not implementation. Specific IdP configuration, token TTLs, and
filter wiring belong to the coding-phase guardrails.

| Aspect | Value |
|---|---|
| **Authentication Required For** | [e.g., All endpoints except `/health` and `/login`] |
| **Identity Source** | [e.g., Corporate SSO / Customer-side SSO / Both / None — internal only] |
| **MFA Expectation** | [None / Admins only / All users] |
| **Service-to-Service Identity** | [e.g., Workload identity / mTLS / Signed JWTs / N/A] |
| **Tenant Isolation Principle** | [e.g., Tenant ID enforced at service layer for every read/write] |
| **Authorization Style** | [e.g., RBAC / ABAC / Policy-as-code / N/A] |

**Authorization model (roles or scopes the design must support):**

| Role / Scope | Intended Access |
|---|---|
| [e.g., `admin`] | [e.g., All endpoints, including admin and audit] |
| [e.g., `user`] | [e.g., Own-tenant data only; no admin endpoints] |
| [e.g., `service`] | [e.g., Internal API surface only] |
| [e.g., `ai-service`] | [e.g., Read-only access to knowledge base; no write or delete] |

<!-- High tier only:
**Sensitive operations requiring step-up auth or re-auth:**
- [e.g., Permanent deletion of customer data]
- [e.g., Role assignment changes]
- [e.g., Export of regulated data]
- [e.g., Enabling autonomous AI agent actions in production]
-->

---

## 6. Network & Exposure Posture

Describe the *surface area* of the system at the design level. Concrete WAF rules,
header values, and rate-limit numbers are tuned during coding and operations.

| Aspect | Value |
|---|---|
| **Exposure** | [Intranet only / Internet-facing / Hybrid (public + internal admin)] |
| **Inbound Surfaces** | [e.g., Public REST API, internal admin API, async event consumer, AI chat endpoint] |
| **Egress Posture** | [Open egress / Allow-list of approved destinations] |
| **Edge Protections Expected** | [e.g., TLS termination, WAF, DDoS protection — required for High tier] |
| **Multi-Tenancy Isolation Layer** | [e.g., Logical (per-row) / Physical (per-cluster) / N/A] |
| **Cross-Region Replication** | [Yes / No — and why] |

---

## 7. Encryption Posture

State **what must be encrypted** and **who owns the keys**. The exact algorithms,
libraries, and field-level encryption tools are coding-phase decisions.

| Aspect | Value |
|---|---|
| **Encryption In Transit** | [Required for all hops / Required only at edge / N/A] |
| **Encryption At Rest** | [Required for all data stores / Required for Sensitive+ only / N/A] |
| **Key Ownership** | [Provider-managed / Customer-managed (CMK) / Customer-held (BYOK/HSM)] |
| **Key Rotation Expectation** | [e.g., Annual automated rotation / Manual / N/A] |
| **Field-Level Encryption Required?** | [Yes — for which classes / No] |

---

## 8. Third-Party & AI Integration Risk

For every external dependency that processes or receives application data, capture
its trust posture here. For AI/LLM providers, the detailed security controls —
prompt injection resilience, output governance, rate limiting, and abuse monitoring —
are specified in **Section 9**.

| Provider | Purpose | Data Shared (Class) | Trust Model | Contractual Controls |
|---|---|---|---|---|
| [e.g., OpenAI] | [e.g., AI enrichment] | [e.g., Domain names — Internal] | [e.g., Untrusted output — see §9] | [e.g., DPA signed, no training opt-in] |
| [e.g., Twilio] | [e.g., MFA SMS] | [e.g., Phone number — Sensitive] | [e.g., Trusted carrier] | [e.g., DPA + SOC 2] |
| [e.g., Stripe] | [e.g., Payments] | [e.g., Payment token — Regulated] | [e.g., PCI-scoped third party] | [e.g., PCI SAQ-A] |

---

## 9. AI / LLM Security Controls

> **Scope:** Complete this section for every service that integrates an AI or LLM
> provider listed in Section 8. If the service has no AI/LLM dependency, mark each
> sub-section `N/A` and skip.
>
> This section captures *design intent*. Concrete guardrail implementations
> (sanitization libraries, moderation API wiring, alerting thresholds) are specified
> in the coding-phase and testing-phase guardrail files.

---

### 9a. Input Security

Controls over what enters the model — covering prompt construction, injection
resilience, and validation of all data before it reaches an AI/LLM surface.

| Control | Required? | Design Intent & Notes |
|---|---|---|
| **Prompt Injection Resilience** | [Yes / No / N/A] | [e.g., System prompt is pinned and immutable at the server; user-supplied content is wrapped in explicit delimiters and clearly marked as untrusted data; no user path to override the system role or escape the instruction boundary] |
| **Jailbreak & Instruction-Override Defense** | [Yes / No / N/A] | [e.g., Read-only or restricted AI surfaces enforce an allow-list of permitted intents; model is instructed to refuse out-of-scope requests; adversarial instruction patterns are detected before reaching the model] |
| **Input Validation & Sanitization** | [Yes / No / N/A] | [e.g., All user-supplied prompt content validated against an allow-schema; tool-call parameters schema-validated before execution; maximum prompt length enforced to prevent context-flooding attacks] |
| **Sensitive Data Scrubbing (Pre-Prompt)** | [Yes / No / N/A] | [e.g., PII, credentials, and internal tokens are detected and redacted from prompt context before the API call is made] |

---

### 9b. Output Governance

Controls over what exits the model — covering factual accuracy posture, content
safety, and validation before AI-generated content reaches users or downstream systems.

| Control | Required? | Design Intent & Notes |
|---|---|---|
| **Hallucination Control** | [Yes / No / N/A] | [e.g., Outputs grounded via RAG with cited, verifiable sources; model is instructed to express uncertainty rather than fabricate; all AI-generated content labeled as AI-generated and not presented as authoritative fact without human review] |
| **Output Safety & Content Policy** | [Yes / No / N/A] | [e.g., Responses screened through a moderation layer before delivery; output checked against enterprise acceptable-use policy, legal requirements, and privacy constraints; refusal and fallback response defined for policy violations] |
| **Output Validation & Sanitization** | [Yes / No / N/A] | [e.g., Structured outputs validated against an expected schema; free-text outputs scanned for PII, secrets, and injection payloads before rendering or storing; AI-generated code reviewed before execution] |
| **Autonomous Action Guardrails** | [Yes / No / N/A] | [e.g., Agent-mode actions require human-in-the-loop confirmation for irreversible operations; blast radius of autonomous actions is bounded; rollback or undo path defined] |

---

### 9c. Operational Controls

Controls over how the AI/LLM integration is operated at runtime — covering resource
abuse prevention, cost governance, observability, and incident response.

| Control | Required? | Design Intent & Notes |
|---|---|---|
| **Rate Limiting** | [Yes / No / N/A] | [e.g., Per-user and per-tenant request-rate limits enforced at the API gateway layer; burst limits set to prevent a single actor from saturating the AI endpoint] |
| **Token Budget & Cost Caps** | [Yes / No / N/A] | [e.g., Maximum prompt + completion token count enforced per request; hard spend cap configured on the provider account; alerts fire before the cap is reached] |
| **Resource Exhaustion Protection** | [Yes / No / N/A] | [e.g., Concurrent AI request limits per service instance; back-pressure applied when queue depth exceeds threshold; timeouts and circuit breakers configured on the provider client] |
| **Monitoring & Anomaly Detection** | [Yes / No / N/A] | [e.g., Prompts and responses logged (with PII redacted) to a SIEM-accessible store; alerts on injection-pattern signatures, unusual token spikes, off-topic intent clusters, and suspected data-exfiltration patterns in outputs] |
| **Abuse Detection** | [Yes / No / N/A] | [e.g., Automated detection of repeated jailbreak attempts, prompt harvesting, and credential-fishing patterns; flagged sessions routed to security review queue] |
| **AI Incident Response** | [Yes / No / N/A] | [e.g., Kill-switch to disable the AI endpoint without a full service deploy; escalation path and on-call rotation defined; abuse playbook linked: [URL]] |

---

## 10. Applied Security Principles

Confirm — at the design level — that each principle has been considered. Note any
deliberate trade-off; do not silently skip.

| Principle | Considered? | Notes / Trade-off |
|---|---|---|
| **Least privilege** (users, services, infra) | [Yes / No] | [e.g., Service runs with a scoped role; no `*` IAM permissions] |
| **Defense in depth** (multiple independent controls) | [Yes / No] | [e.g., Auth at edge AND service layer; AI outputs validated at model boundary AND before rendering] |
| **Fail secure / fail closed** | [Yes / No] | [e.g., On auth-service outage, deny requests rather than allow; on AI provider outage, surface a safe fallback — not raw error detail] |
| **Separation of duties** | [Yes / No] | [e.g., Prod deploys require two approvers] |
| **Minimize attack surface** | [Yes / No] | [e.g., Internal endpoints not exposed via public ALB; system prompt never returned in API responses] |
| **Secure by default** | [Yes / No] | [e.g., New tenants start with strictest permissions; AI features disabled until explicitly enabled] |
| **Do not trust the client / zero trust** | [Yes / No] | [e.g., All trust decisions made server-side] |
| **Auditability by design** | [Yes / No] | [e.g., Every sensitive action emits a structured audit event; AI interactions logged for traceability] |
| **Data minimization** | [Yes / No] | [e.g., Only fields needed for the feature are collected; prompts contain the minimum context required] |
| **Privacy by design** (if PII) | [Yes / No] | [e.g., PII pseudonymized in analytics pipeline; PII scrubbed before reaching AI/LLM provider] |
| **AI output treated as untrusted** | [Yes / No] | [e.g., All model responses validated, sanitized, and labeled before downstream use — regardless of provider trust level] |
| **AI cost and quota bounded** | [Yes / No] | [e.g., Spend cap, token budget, and rate limits in place; no unbounded resource paths reachable by end users] |

---

## 11. Open Risks, Assumptions & Exceptions

Capture the things you knowingly chose *not* to handle in this design — with
business justification. Exceptions belong here, not buried in code.

| Item | Type (Risk / Assumption / Exception) | Owner | Justification & Compensating Control |
|---|---|---|---|
| [e.g., No DDoS protection on internal admin API] | Exception | [@owner] | [e.g., Intranet-only; mitigated by VPN-only ingress] |
| [e.g., Assumes upstream IdP enforces MFA] | Assumption | [@owner] | [e.g., Verified by platform team Q1] |
| [e.g., Field-level encryption deferred] | Risk | [@owner] | [e.g., Will revisit if data class upgraded to Regulated] |
| [e.g., AI output moderation relies solely on provider-side filtering] | Risk | [@owner] | [e.g., Accepted for MVP; application-layer moderation will be added before GA] |
| [e.g., Hallucination control not implemented for internal search assistant] | Exception | [@owner] | [e.g., Low-stakes internal tool; outputs labeled as AI-generated; re-evaluate if scope expands to customers] |

---

## 12. Hand-off to Later SDLC Phases

This template is the *entry* to the security workflow, not the end of it. Confirm
the hand-offs below so the downstream phases have what they need.

| Next Phase | Status / Link |
|---|---|
| **Threat Modeling** (consumes this doc; produces detailed mitigations, including AI-specific abuse cases) | [e.g., Triggered — link to threat-model output / Not yet started] |
| **Coding-Phase Guardrails** (stack-specific MD applied during development) | [e.g., `security-coding-java.md` / `security-coding-python.md` / TBD] |
| **AI/LLM Coding Guardrails** (prompt construction, output handling, provider SDK usage) | [e.g., `security-coding-llm.md` / TBD / N/A] |
| **Testing-Phase Guardrails** (SAST/SCA/DAST/pentest gating per tier) | [e.g., Standard CI suite + DAST for Moderate / Pen-test + AI red-team required for High] |
| **Security Sign-off** | [Required for High tier — Name / Date] |

---

## Glossary

| Term | Definition |
|---|---|
| **CIA Triad** | Confidentiality, Integrity, Availability — the three classic pillars of security. |
| **Trust Boundary** | Any point at which the level of trust in data or callers changes. |
| **Tenant Isolation** | Mechanism that prevents one customer's data or actions from affecting another's. |
| **CMK** | Customer Managed Key — encryption key owned and controlled by the application's organization. |
| **RBAC / ABAC** | Role-Based / Attribute-Based Access Control. |
| **Defense in Depth** | Layered security so that the failure of any single control does not lead to compromise. |
| **Fail Secure** | Design choice in which failure of a component results in denial rather than permission. |
| **Threat Modeling** | Structured analysis of how a system could be attacked, producing per-asset mitigations. |
| **Prompt Injection** | Attack in which malicious user-controlled text overrides or hijacks the AI model's instructions. |
| **Jailbreak** | Adversarial technique to bypass an AI model's safety constraints or system-level instructions. |
| **Hallucination** | AI model behavior in which plausible-sounding but factually unsupported content is generated and presented as fact. |
| **Grounding** | Technique (e.g., RAG) that anchors model responses to a verified, retrievable knowledge source to reduce hallucination. |
| **RAG** | Retrieval-Augmented Generation — pattern in which relevant documents are retrieved and injected into the prompt context before model inference. |
| **System Prompt** | The privileged instruction block provided to an LLM by the application (not the end user) that defines the model's behavior, scope, and persona. |
| **Token Budget** | Maximum number of tokens (prompt + completion) permitted per AI request, used to control cost and prevent resource exhaustion. |
| **Output Moderation** | Automated screening of model outputs against safety, content policy, and privacy rules before delivery to users or downstream systems. |
| **AI Red-Team** | Structured adversarial testing of an AI system to discover prompt injection paths, policy bypasses, and unsafe outputs. |
