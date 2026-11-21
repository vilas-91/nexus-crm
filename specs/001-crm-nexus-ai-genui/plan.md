# Implementation Plan: CRM Nexus AI Platform & GenUI

**Branch**: `001-crm-nexus-ai-genui` | **Date**: 2026-11-21  
**Spec**: `specs/001-crm-nexus-ai-genui/spec.md`  
**Canonical domain reference**: `specs/1-domain-model/spec.md`

## Summary

Deliver the six approved V1 AI capabilities—Lead Summary, Lead Scoring, Notes Summarization,
Email Generation, Ask CRM, and AI History—inside the CRM Nexus tenant boundary. Use Next.js App
Router, TypeScript, HeroUI, Tailwind CSS, Supabase PostgreSQL/Auth/RLS, pgvector, and
CopilotKit-compatible GenUI. Implement business behavior in independently testable domain
services behind repository, provider, embedding, secret-store, and audit ports. Server Actions,
route handlers, CopilotKit adapters, and a future MCP adapter are thin, validated transports
over those services.

Generated AI content is a reviewable result, never authority to mutate CRM data. An exact
proposal must be explicitly approved by a same-tenant authorized user; application rechecks
membership, permission, subject revision, and proposal identity. Provider failures, rejection,
expiry, cancellation, and audit failures never become success-shaped CRM changes. RAG uses only
current, ready, attributable evidence eligible for the requesting user.

This plan does not implement a public REST API or MCP server, external email delivery, a
workflow engine, or CRM lifecycle changes. It assumes the Foundation Platform supplies active
tenant identity, membership, permissions, audit infrastructure, and shell, and CRM Core supplies
the canonical CRM services and records.

## Technical Context

**Language/Version**: TypeScript; use the latest stable Node.js/Next.js supported by Vercel at implementation start and pin exact versions in the lockfile.  
**Primary Dependencies**: Next.js App Router, React, HeroUI v3, Tailwind CSS v4, CopilotKit React/runtime, Supabase SSR/client libraries, Zod, Vitest, Playwright, and aimock; provider SDKs are confined to server-side adapters and are not dependencies of domain services.  
**Storage**: Supabase PostgreSQL for canonical entities, audit, AI history, prompts, knowledge metadata, and pgvector embeddings; Supabase Auth for identity; Supabase Storage only for authorized source files already covered by the platform.  
**Testing**: Vitest unit and integration suites; PostgreSQL/Supabase migration and RLS tests; aimock or deterministic provider fakes; Playwright user journeys and responsive/accessibility checks.  
**Target Platform**: Vercel-hosted Next.js application and Supabase-managed backend; server-only provider integrations and optional asynchronous ingestion execution.  
**Project Type**: Multi-tenant web application with Next.js full-stack monolith and explicit
domain/transport/provider boundaries.  
**Performance Goals**: Meet feature targets: start loading/cancel feedback immediately and show
stream/progress within 2 seconds where supported; Ask CRM p95 ≤10 seconds; summaries, drafts,
and scoring p95 ≤15 seconds; eligible RAG retrieval p95 ≤1 second; latest 100 history rows ≤2
seconds. Measure provider latency separately from retrieval/service latency.  
**Constraints**: Authenticated tenant context must be revalidated server-side; RLS and application
authorization are both required. AI usage, history, and audit are append-oriented. No hidden
provider fallback. All external/persisted trust-boundary values are runtime-validated. Secrets,
raw prompts, and unnecessary personal data must not enter audit/log payloads.  
**Scale/Scope**: Six V1 AI capabilities; CRM Core's planning baseline is up to 100,000 active
records per core CRM entity and 50 concurrent users per Tenant. The AI specification leaves
generation/knowledge volume to deployment planning; capacity tests must therefore use an
explicitly recorded V1 workload profile before production.

### Planning assumptions and decisions

1. Foundation Platform and CRM Core are implementation dependencies, not duplicated features.
   The plan calls their authenticated domain services and canonical entities; if they do not
   exist at implementation start, their approved plans/tasks must be delivered first.
2. The selected tenant is untrusted transport input. Each adapter derives the user from the
   authenticated server session and asks the platform authorization service to resolve and
   validate the active membership before passing an `ActorContext` to a domain service.
3. Use a user-scoped Supabase server client for tenant business access. Do not expose a
   service-role key to browsers or use it for ordinary user requests. Any system ingestion job
   must use a narrowly scoped server-only entry point with explicit tenant and source checks.
4. Persist provider identity, permitted model aliases, capability settings, and a non-secret
   secret reference only. Resolve credential values through a server-only `SecretStore` port;
   never place credentials in ordinary provider attributes, prompts, audit, errors, or logs.
5. Bind each embedding to an explicit model/revision and dimension. A deployment selects and
   pins its V1 embedding model/dimension before creating the vector index; a model/dimension
   change creates a new revision/index path and never reinterprets old vectors.
6. Store proposal/review state in canonical `AIGeneration` and append-only `AuditLog` facts;
   do not introduce a competing proposal entity. Persist or reference only the minimum output
   allowed by tenant retention policy.
7. There are no unresolved product questions in the feature spec. The implementation team must
   finalize deployment policy inputs (secret-store backend, embedding model/dimensions, retention,
   V1 AI workload/quotas) before enabling production ingestion or accepting tenant credentials.

## Constitution Check

**Pre-research gate: PASS, subject to the explicit governance/dependency gates below.** The
user identifies the AI Platform & GenUI specification as approved; its file header still says
`Draft`. The plan treats the user's approval as authoritative for planning and records the
metadata mismatch as an implementation precondition.

| Required check | Result | Plan evidence |
|---|---|---|
| AI-Native Product | PASS | Six defined capabilities; consequential application requires explicit approval; no autonomous CRM mutation. |
| Multi-Tenant By Default | PASS | Authenticated active membership, service authorization, same-tenant references, scoped retrieval/cache, PostgreSQL RLS, and isolation tests. |
| Next.js Architecture | PASS | App Router; Server Components for read surfaces; narrowly scoped Client Components for GenUI; Actions/Routes are adapters. |
| Type Safety | PASS | TypeScript throughout; Zod at transport, capability, provider, proposal, retrieval, and persistence boundaries. |
| Provider-Agnostic AI | PASS | Provider registry/port with OpenAI, Gemini, Ollama, and OpenRouter adapters; no domain dependency on vendor SDKs or hidden fallback. |
| RAG-Ready | PASS | Canonical KnowledgeDocument/Chunk/VectorEmbedding, Supabase pgvector, revision invalidation, authorization before ranking, replaceable embedding adapter. |
| Transport-Agnostic Services | PASS | Domain services are transport-independent and reusable from Actions, routes, CopilotKit, and a future MCP adapter. |
| AI UI Compatibility | PASS | CopilotKit renders tool/proposal/progress/review states; approval is separately enforced by the domain service. |
| Mobile and Accessibility | PASS | Mobile-first, keyboard-operable, semantic, WCAG 2.2 AA-oriented surfaces and Playwright checks. |
| Testing | PASS | Vitest, database/RLS tests, deterministic AI mocks/aimock, and Playwright. |
| Specification-Driven Development | PASS | Functional requirements and acceptance criteria map to architecture, slices, tests, and `tasks.md`. |
| Domain Model | PASS | Reuses canonical entity names, ownership, lifecycle, revision, soft-delete, and audit rules; no new domain entity. |
| Theme Compatibility | PASS | Shared scalable tokens; light and dark themes for all AI and review surfaces. |
| Preferred Stack / V1 Scope | PASS | Uses the mandated stack and excludes payment, WhatsApp, voice, automation engine, marketplace, multi-region, multi-language, and MCP deployment. |

No constitutional deviation is proposed; complexity tracking is not applicable.

### Governance and cross-spec gates

- **G-1 — Approval metadata**: `specs/001-crm-nexus-ai-genui/spec.md` declares `Draft`, despite
  the requester identifying it as approved. Confirm/record the approval status before
  implementation; planning proceeds on the supplied approval assertion.
- **G-2 — Canonical/foundation approval**: the domain-model document is labeled
  “Draft — canonical upon approval”; Foundation Platform and CRM Core also state domain-model
  approval as an implementation prerequisite. Formal approval of the domain model and those
  prerequisite specifications is required before implementation, as specified by their own
  headers. This plan uses the domain-model text as the requested canonical reference.
- **G-3 — Note-AI boundary**: Resolved by CRM Core FR-032. V1 may provide only an on-demand,
  transient Note Summary for an authorized current Note revision. Return summary text only in the
  request response; retain trace metadata and source Note revision only. Do not index or embed
  Note content, use Notes as retrieval sources, or use Notes in other V1 AI capabilities. Reject
  stale/deleted revisions. Any future derived artifact must retain that revision and become stale
  when it changes or is deleted.
- **G-4 — Deployment policy**: choose a production secret-store backend, embedding model and
  dimensions, retention policy, and workload/usage quotas before storing tenant credentials,
  indexing production content, or enabling production traffic. These are deployment-policy
  inputs explicitly deferred by the feature spec, not unresolved product requirements.

These are tracked implementation gates, not failures of the architecture or unresolved
specification clarifications.

## Architecture and Design

### Layering and request flow

```text
Next.js App Router UI (Server Components + focused Client Components)
    ├── Server Actions / Route Handlers / CopilotKit runtime adapter
    │       └── Zod input validation + authenticated ActorContext resolution
    └── Transport-agnostic application/domain services
            ├── CRM read/write/search ports → existing CRM Core services
            ├── AI orchestration → ProviderRegistry → provider adapters
            ├── Prompt / review / history / usage / audit services
            ├── Knowledge ingestion / RetrievalService → embedding port
            └── repositories → Supabase PostgreSQL / pgvector under RLS
```

Transport code performs protocol conversion, validation, authentication-context resolution,
response shaping, and UI cache invalidation only. It must not implement permissions, CRM rules,
prompt selection, provider fallback, approval, or RAG eligibility. Domain services accept typed
commands and an authenticated `ActorContext`; that context is created server-side and is never
trusted merely because a caller supplied a Tenant ID.

### Persistence, authorization, and audit

- Use versioned SQL migrations as the schema/policy source of truth. Enable RLS on every
  exposed tenant-owned table, set explicit grants as well as policies, and test both. Include
  `tenant_id` on tenant-owned records and enforce same-tenant foreign-key relationships where
  supported (composite tenant/id keys or equivalent constraints).
- RLS policies derive access from the authenticated identity and active
  `UserRoleAssignment`/permission state. Application services also check active Tenant,
  membership, capability permission, record permission, and revision; neither layer replaces
  the other. Avoid authorization based only on navigation, feature flags, or client state.
- Make critical state change + required audit outcome atomic through a narrow Postgres function
  or transaction-capable repository operation. If audit cannot be durably appended, fail the
  critical operation and return an explicit failure. Audit and usage corrections are
  compensating records, not in-place edits.
- Keep secrets outside ordinary tables and responses. Do not log request/response bodies by
  default. Apply retention/redaction policy to prompt and model output references.
- Use tenant-, user-permission-, source-revision-, and lifecycle-aware cache keys; default AI
  and RAG responses to non-cacheable unless all authorization inputs are safely represented.

### AI provider and prompt services

Define a stable provider port for capability execution and embedding. Adapters normalize
provider-specific model IDs, streaming, usage, timeouts, cancellation, and errors into
vendor-neutral contracts. The registry selects only a Tenant-enabled provider/model allowed for
the capability. No implicit cross-provider retry: a retry uses the selected provider under
documented retry/idempotency rules; changing providers requires explicit policy/selection.
Provider output and structured tool arguments are Zod-validated before persistence or display.

Resolve the exact immutable active `AIPromptTemplate` version at request time and retain that
reference in `AIGeneration`. Capability contracts constrain inputs, permitted source types,
output shape, and side effects. CRM/RAG text is untrusted data and cannot grant tools or override
system policy. Minimization, source attribution, and refusal/insufficient-evidence behavior are
part of the service contract rather than post-processing UI conventions.

### RAG and knowledge lifecycle

Knowledge registration creates/revises canonical `KnowledgeDocument`; an ingestion job builds
`KnowledgeChunk` records and `VectorEmbedding` entries with source revision, embedding-model
revision, and dimension metadata. Use pgvector in Supabase PostgreSQL. Similarity search runs
through a server-side Postgres function/RPC that applies tenant, permission/source eligibility,
document/chunk/embedding lifecycle, revision, and deletion predicates before ranking/limiting.
Do not rely on a global vector query followed by application filtering. Only `ready` vectors
from current, indexed, active sources can be returned. Edits/archive/delete mark old artifacts
stale/ineligible immediately, independently of asynchronous cleanup.

The embedding adapter and model metadata are replaceable; pin the V1 model/dimension as a
deployment/migration setting before creating the HNSW/IVFFlat index. Validate RAG quality,
thresholds, and retrieval latency with tenant-isolated fixtures. Notes are never eligible
`KnowledgeDocument` sources or retrieval evidence in V1. Note Summary uses only the exact,
authorized current Note revision and returns transient output with revision attribution.

### CopilotKit and GenUI

CopilotKit is the interactive presentation/streaming adapter, not an authorization or domain
layer. Provide explicit states for requested, streaming/progress, succeeded, failed, cancelled,
awaiting review, approved, rejected, and expired. Render citations and source scope; allow
editing an email draft locally before separate save/send intent. The V1 experience does not
send email externally.

An approval control submits the exact generation/proposal reference and expected subject
revision to the approval service. The service revalidates the actor, active membership,
permission, proposal hash/revision, subject revision, and provider/policy state, then applies
through an existing CRM domain command exactly once and appends the approval/application audit
outcome. CopilotKit tool response, user confirmation, or hidden agent state alone is never
authorization.

### Vertical feature slices

Every slice includes its migration/policy, domain service and ports, transport adapter, UI,
audit/history, unit/integration/RLS tests, Playwright scenario, and docs as applicable.

1. **S0 — Shared AI foundation**: migration/test scaffolding; request context, authorization,
   provider/secret/embedding/audit/repository ports; capability and result schemas; mock runtime.
2. **S1 — Provider and prompt governance**: tenant provider configuration/health, secret
   reference handling, supported adapters, versioned prompt templates, admin UI, audit, and
   provider failure/no-fallback tests.
3. **S2 — Lead Summary and Score**: permission-scoped CRM context, attributed sources,
   validated summary/score, CopilotKit/record-level review surfaces, generation trace/usage,
   and no-mutation tests.
4. **S3 — Notes Summary and Email Draft**: transient on-demand summaries of authorized current
   Note revisions with exact revision attribution and no persistence/indexing; editable draft for
   Lead/Contact/Deal; separate explicit approval for persistence and no V1 external send.
5. **S4 — Knowledge registration, ingestion, and Ask CRM**: tenant-owned source lifecycle,
   chunk/embedding revisions, stale invalidation, pgvector RPC, evidence-grounded answers,
   citations, insufficient-evidence behavior, and cross-tenant RAG tests.
6. **S5 — AI History, approval, and operations**: authorized history views, review decisions,
   revalidation/idempotent application, usage reconciliation, append-only corrections,
   cancellation/failure diagnostics, and admin governance.

Slices S1/S0 are prerequisites for all AI capabilities. S2/S3 can proceed independently after
S0/S1; S4 precedes RAG-backed Ask CRM; S5 cross-cuts the other slices and completes their
governance surfaces. The exact delivery order is recorded in `tasks.md`.

## Implementation Phases

### Phase 0 — Research and decision baseline (complete in this plan)

Resolve architecture and technology choices against official documentation; record RLS,
server-action, pgvector, CopilotKit, and design-system boundaries in `research.md`.

### Phase 1 — Design and contracts (complete in this plan)

Define canonical entity use and transitions in `data-model.md`, reusable service/adapter
contracts in `contracts/`, a local validation path in `quickstart.md`, and update Copilot agent
context. No business entity meaning or relationship is changed.

### Phase 2 — Implementation execution (planned; not performed here)

1. Establish/apply prerequisites and shared foundation (G-1 through G-4).
2. Build S0/S1 and verify tenant policy, provider/secret boundaries, audit, and mocks.
3. Deliver S2/S3/S4/S5 as end-to-end vertical slices, then harden latency/accessibility.
4. Run staging verification, tenant-isolation suite, migration rehearsal, and controlled rollout.

### Phase 3 — Deployment and operation (planned; not performed here)

Deploy the Next.js application to Vercel and database migrations/extension/policies to Supabase.
Run migration and RLS checks in a disposable Supabase project before staging/production;
configure server-only credentials and environment-specific provider/embedding settings; verify
health/telemetry, budgets, retention, quotas, cancellation, and rollback. Enable features by
tenant only after admin configuration and staged isolation/performance gates pass. Feature flags
gate availability only and never grant access.

## Testing and Quality Strategy

- **Domain unit tests (Vitest)**: approval invariants, lifecycle transitions, capability
  contracts, provider selection, prompt version resolution, source attribution, failure and
  cancellation behavior, idempotency, and audit-required failure semantics.
- **Port/service integration tests**: repository and provider fakes; no live provider dependency.
  Use aimock where supported; otherwise deterministic test adapters cover the same port.
- **Database and RLS tests**: migrations, explicit grants, RLS for every exposed table, same-
  tenant foreign keys, multi-tenant users, forged/stale tenant context, cross-tenant reads/writes,
  history and audit, and retrieval eligibility. Test signed-in user and tightly scoped system
  ingestion roles separately; prove service-role paths cannot escape the server boundary.
- **RAG tests**: ready/current/authorized non-Note results only; deny Note sources as well as
  pending, failed, stale, deleted, archived, cross-tenant, and unauthorized parent sources.
  Test transient Note Summary revision attribution and stale/deleted-source rejection, citation
  mapping, no-evidence response, bounded result count, and RPC latency.
- **Provider tests**: supported adapters contract-test normalized outputs, streaming, timeout,
  cancellation, malformed/disallowed output, disabled provider, missing credential, usage
  capture, and no silent fallback. Secrets must be absent in captured payloads and logs.
- **Browser tests (Playwright)**: all six capabilities; admin provider/template and knowledge
  configuration; history/review; approved vs rejected/expired/cancelled; two-tenant user;
  keyboard-only operation; mobile/tablet/desktop; light/dark; visible loading/failure states.
- **Accessibility**: automated checks plus keyboard/manual acceptance of focus order, semantic
  announcements, labels, contrast, dialogs, citations, and review controls at WCAG 2.2 AA
  orientation.
- **Performance and operational tests**: record retrieval and provider latency separately;
  verify 1s RAG retrieval p95, 10s Ask CRM p95, 15s summary/score/draft p95, first progress
  within 2s where supported, and 2s latest-100 history. Establish AI workload/concurrency
  baseline before sign-off. Exercise provider outage, audit failure, queue retry, and migration
  rollback.

## Deployment and Observability

- **Vercel**: production Next.js App Router deployment; server-only environment configuration;
  preview environments must use isolated Supabase projects or test data, never production
  credentials. Keep CopilotKit runtime and provider SDK calls server-side. Configure request
  body/streaming/time limits and cancellation behavior for deployment constraints.
- **Supabase**: ordered SQL migrations; enable pgvector extension; explicit role grants and RLS;
  indexes for tenant/state/history lookups and a measured vector index; least-privilege
  credentials; backups and point-in-time recovery according to retention policy.
- **Ingestion**: use a bounded, retryable background execution mechanism selected from available
  deployment facilities. Jobs are idempotent and tenant/source/revision-scoped; stale state is
  committed before queueing cleanup/reindex work.
- **Observability**: correlation IDs across transport, service, provider, ingestion, audit, and
  usage. Measure latency/outcome, token/usage when returned, retrieval count, and queue delay.
  Redact credentials, tokens, raw prompts, raw model output, and unnecessary personal content.
  Provider diagnostics use normalized categories, not vendor payload dumps.
- **Rollout**: migrations first; shadow/fixture RAG checks; internal tenant; limited tenant
  enablement; verify tenant isolation, latency, audit, provider health, and cost/usage; expand
  only after thresholds pass. Roll back by disabling capability/ingestion and reverting
  application version; never rewrite append-only AI/audit facts to roll back.

## Risks and Mitigations

| Risk | Mitigation / verification |
|---|---|
| Tenant or permission context omitted in an adapter, cache, or retrieval path | Resolve actor server-side, validate membership per operation, enforce RLS, tenant-aware cache keys, negative two-tenant tests. |
| Supabase service-role bypasses RLS | Restrict key to server secrets; avoid for ordinary operations; narrowly scoped system functions and explicit tenant/source checks; test separately. |
| Approval becomes a UI-only confirmation | Domain approval service checks exact proposal/revision and actor, revalidates record/permission, applies once, and writes audit outcome. |
| Note content enters V1 retrieval or becomes a durable summary | Enforce FR-032 in source registration, embeddings, retrieval RPC, and capability contracts; only transient on-demand Note Summary is allowed, bound to the exact source revision. |
| Provider quality, outage, timeout, or malicious output | Normalize failure, schema-validate, make no hidden fallback, show failure/retry/cancel, keep CRM unchanged. |
| Prompt injection from retrieved content | Mark retrieved text as untrusted evidence, restrict tool authority, limit sources, validate response, and never rely on model refusal for authorization. |
| Stale or weak RAG evidence | Revision/state invalidation before async work; filter eligibility before vector ranking; cite source revisions; state insufficient evidence; evaluate threshold and latency. |
| Secrets or sensitive content leak into storage/logs | Server-only secret-store port, secret references only, minimize prompt/output persistence, structured redaction tests, retention gate. |
| Append-only history growth and cost | Define retention/quotas before production; use pagination and compensation; measure provider usage; never silently rewrite facts. |
| Provider/model or embedding migration changes semantics | Version provider/model/template/embedding revisions; pin vector dimension per index; explicit rollout/reindex and provenance. |
| Context window/Next.js request limits disrupt streaming | Bound source/context size, cancellation-aware provider port, progress UI, deployment integration tests and transparent truncated/failure states. |
| Foundation/CRM dependencies or approval statuses are not ready | Enforce gates G-1/G-2; do not duplicate or bypass their services; schedule prerequisites ahead of S0. |

## Traceability

| Requirement area | Planned slice / evidence |
|---|---|
| FR-001–008, AC-001, AC-006: capabilities, summary/score, source attribution | S2/S3/S4; capability contracts; service and browser tests |
| FR-009, FR-012–014: email draft, approval and non-application | S3/S5; exact proposal/review contract; approve/reject/expiry tests |
| FR-002–004, FR-015–016, FR-020–023: trace, providers, prompts, audit, usage, reusable services | S0/S1/S5; data model and ports; provider/audit tests |
| FR-005, FR-010, FR-017–019, FR-029: scoped retrieval, citations, stale/deleted exclusion | S4; retrieval contract; pgvector/RLS/revision tests |
| FR-011, FR-013–014, FR-021–022: append-only history, review, usage correction | S5; immutable record model; compensation and audit tests |
| FR-024–028: CopilotKit, mocking, validation, responsive accessible UI, explicit failures | All slices; GenUI contract; Vitest, Playwright, accessibility tests |
| FR-030, AC-010: future REST/MCP service reuse without deployment | S0 ports and adapter contract; contract-level reuse test; no V1 MCP server |
| NFR/SC: latency, isolation, traceability, accessibility, themes | Quality strategy, staging deployment checklist, `tasks.md` verification tasks |

## Project Structure

```text
specs/001-crm-nexus-ai-genui/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── tasks.md
└── contracts/
    ├── ai-domain-contracts.md
    └── transport-adapters.md

src/
├── app/
│   ├── (auth)/...
│   ├── (crm)/[tenantSlug]/ai/...
│   ├── api/copilotkit/route.ts
│   └── actions/...
├── features/
│   ├── ai-history/
│   ├── ai-settings/
│   ├── ask-crm/
│   ├── email-draft/
│   ├── knowledge/
│   ├── lead-ai/
│   └── notes-ai/
├── components/
│   ├── genui/
│   ├── shell/
│   └── ui/
├── domain/
│   ├── ai/
│   ├── authorization/
│   ├── knowledge/
│   └── ports/
├── infrastructure/
│   ├── ai-providers/
│   ├── embeddings/
│   ├── secrets/
│   └── supabase/
│       ├── repositories/
│       └── server/
└── shared/
    ├── schemas/
    ├── errors/
    └── observability/

supabase/
├── migrations/
├── functions/                 # narrow SQL/RPC functions, not business transport handlers
└── tests/                     # migration and RLS/database tests

tests/
├── integration/
├── e2e/
├── fixtures/
└── support/
```

**Structure decision**: Next.js web application organized by vertical feature slices, with
transport-agnostic domain services and explicit infrastructure adapters. `src/app` owns routing
and adapters; feature folders own UI composition; domain services own business behavior;
infrastructure owns Supabase, providers, embeddings, and secret access. Canonical CRM behavior
remains in CRM Core and is called through its service boundary.
