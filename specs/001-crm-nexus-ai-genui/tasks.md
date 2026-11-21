# Implementation Tasks: CRM Nexus AI Platform & GenUI

This is the actionable implementation backlog derived from `plan.md`. It is not executed by the
planning task. Tasks are sequenced by actual prerequisites; each vertical slice includes its
schema/policies, domain behavior, adapter/UI, and verification.

## Phase 0 — Resolve implementation gates and bootstrap

- [ ] **T001 — Confirm approval baseline**: Update the target spec approval metadata or record
  the product approval decision; approve the canonical domain model and prerequisite Foundation
  Platform/CRM Core specifications as required by their headers. **Blocks implementation**
  (G-1, G-2).
- [ ] **T002 — Enforce the Note-AI boundary**: Implement only transient on-demand Note Summary
  for authorized current revisions; prohibit summary persistence, Note registration/indexing/
  embeddings, Note retrieval, and Note use by other V1 AI capabilities. Cite the source revision
  and reject stale/deleted revisions. Ensure future derived artifacts bind to that revision and
  become stale on change/deletion. **Depends on T001** (FR-008, FR-019).
- [ ] **T003 — Approve deployment policy inputs**: Select a secret-store backend, V1 embedding
  model/revision/dimension, prompt/output retention policy, AI workload profile/quotas, and
  ingestion execution facility. Document rotation, reindex, retention, and rollback behavior.
  **Depends on T001** (G-4).
- [ ] **T004 — Bootstrap the application**: Initialize the Next.js App Router TypeScript app,
  strict typechecking, lockfile, HeroUI v3/Tailwind v4 themes, CopilotKit runtime, Supabase
  server integration, Zod, Vitest, Playwright, and test scripts. Establish the folder structure
  in `plan.md`.
- [ ] **T005 — Establish local environments**: Add environment-variable names to an ignored
  local template, local Supabase CLI configuration, disposable test project, two-tenant seed
  fixtures, and CI checks. Confirm no secrets are committed or bundled client-side.
- [ ] **T006 — Add canonical migrations and authorization baseline**: Implement migrations for
  required existing platform/CRM prerequisites or verify their approved migrations are present;
  add only AI tables/columns/indexes required by the canonical model. Add same-tenant references,
  explicit grants, RLS on every exposed table, and migration/RLS tests. **Depends on T001**.
- [ ] **T007 — Implement authenticated actor context**: Resolve active Tenant membership and
  permissions from server session; reject forged/stale tenant selectors; define shared domain
  errors and request correlation. Revalidate context on every service boundary. **Depends on
  T004, T006**.

## Phase 1 — Shared service and infrastructure foundation (S0)

- [ ] **T010 — Define domain ports and schemas**: Implement typed/Zod contracts for ActorContext,
  capabilities, subject refs/revisions, citations, generation results, provider requests,
  normalized errors, idempotency, and pagination from `contracts/ai-domain-contracts.md`.
  **Depends on T007** (FR-023, FR-025–026).
- [ ] **T011 — Implement repository and audit ports**: Add tenant-scoped repositories for
  canonical AI records and append-only `AuditLog`/`AIUsageRecord`; ensure required audit outcome
  is atomic with or a fail-closed prerequisite to critical state changes. Test compensation,
  uniqueness, and duplicate idempotency. **Depends on T006, T010** (FR-002, FR-011, FR-020–022).
- [ ] **T012 — Implement provider/secret/embedding ports**: Define provider registry, normalized
  completion/stream/cancellation/error behavior, embedding model metadata, and server-only
  SecretStore resolution; keep all SDKs and credentials out of domain modules. **Depends on
  T003, T010** (FR-003, FR-015–016, FR-025).
- [ ] **T013 — Implement shared test adapters**: Add deterministic provider, embedding,
  repository, authorization, audit, and secret-store fakes plus aimock integration where
  supported. Make unit/integration suites run without network credentials. **Depends on T010–T012**
  (FR-025, AC-008).

## Phase 2 — Provider and prompt governance (S1)

- [ ] **T020 — Implement tenant AI provider configuration**: Add authorized provider
  registration/enable/disable/retirement, capability/model allow-lists, health state, secret
  reference handling, and audit. Do not return or log credential values. **Depends on T011–T013**
  (FR-015–016, FR-020).
- [ ] **T021 — Add provider adapters**: Implement server-only OpenAI, Gemini, Ollama, and
  OpenRouter adapters behind the common port; normalize model IDs, usage, streaming,
  cancellation, timeouts, and failures. Enforce explicit provider selection/no hidden fallback.
  **Depends on T012, T020** (FR-003, FR-028).
- [ ] **T022 — Implement prompt-template versioning**: Create/publish tenant-owned templates
  with immutable active versions, declared input/output contracts, exact generation provenance,
  and audit. **Depends on T011, T020** (FR-004).
- [ ] **T023 — Build provider/template settings UI**: Create accessible responsive admin forms,
  safe health status and template version history in light/dark themes; show secret configured
  state without exposing values. **Depends on T020–T022**.
- [ ] **T024 — Verify governance boundaries**: Test cross-tenant configuration denial, disabled
  provider, invalid credential, malformed output, timeout/cancellation, no fallback, immutable
  prompt version, and secret redaction using unit, database, and browser tests. **Depends on
  T020–T023** (AC-002, AC-005, AC-008).

## Phase 3 — Approval and trace foundation (S5 foundation)

- [ ] **T030 — Implement generation lifecycle service**: Create pending `AIGeneration`, persist
  validated success/failure/cancellation and provenance, capture provider/model/template
  metadata, and append usage records when returned. **Depends on T011–T013, T020–T022**
  (FR-002, FR-021).
- [ ] **T031 — Implement approval service**: Store review state using canonical generation and
  audit records; authorize exact proposal revision; revalidate membership, permission, subject
  revisions, and idempotency; call existing CRM command only after approval; fail closed on audit
  failure. **Depends on T007, T011, T030** (FR-012–014).
- [ ] **T032 — Implement history query service**: Add permission-filtered, paginated history
  queries for requester/provider/model/status/review state and bounded latest-100 reads.
  **Depends on T030** (FR-011, FR-020–022).
- [ ] **T033 — Test approval and trace invariants**: Verify rejected, expired, failed, cancelled,
  stale permission/revision, duplicate approval, provider failure, and audit failure never mutate
  CRM; verify append-only correction and attributable outcomes. **Depends on T031–T032**
  (AC-003–005).

## Phase 4 — Lead Summary and Lead Scoring (S2)

- [ ] **T040 — Implement Lead context and summary capability**: Use CRM Core read services for
  same-tenant permitted Lead/related context, minimize fields, resolve prompt/provider, validate
  output, create trace, and return source citations without mutation. **Depends on T030** (FR-005–006).
- [ ] **T041 — Implement Lead scoring capability**: Validate score range/factors and evidence;
  ensure score is explanatory and cannot update canonical Lead fields. **Depends on T040**
  (FR-007).
- [ ] **T042 — Build Lead GenUI**: Add record-level actions, progress/stream/cancel states,
  attributed result cards, failure/insufficient-data states, and no-mutation affordances with
  HeroUI/CopilotKit. **Depends on T040–T041** (FR-024, FR-027–028).
- [ ] **T043 — Verify Lead AI slice**: Add two-tenant/service permission tests and Playwright
  scenarios for summary, scoring, citations, no permission, cancellation, provider failure,
  keyboard/responsive/light/dark behavior. Confirm Lead revision is unchanged. **Depends on
  T042** (US1, AC-001–002, AC-006, AC-009).

## Phase 5 — Notes Summary and Email Draft (S3)

- [ ] **T050 — Implement transient Note summarization**: Load only the accessible current Note
  revision; return summary text only in the request response and persist trace metadata/source
  revision only; do not use Note content in another capability. Return the exact Note revision
  citation and reject stale/deleted revisions. **Depends on T002, T030** (FR-008).
- [ ] **T051 — Implement email draft generation**: Support Lead/Contact/Deal context under
  subject permissions; return editable draft, keep draft unapplied, and omit external delivery.
  **Depends on T030** (FR-009).
- [ ] **T052 — Connect email save intent to approval**: If the user explicitly requests CRM
  persistence, create an exact proposal and call approval/domain application service; prohibit
  implicit save and any V1 send integration. **Depends on T031, T051** (FR-009, FR-012–014).
- [ ] **T053 — Build Notes/email GenUI**: Add attributed Note summary and editable draft views;
  label draft vs saved state, provide explicit separate confirmation, accessible progress/errors,
  and mobile/light/dark layouts. **Depends on T050–T052**.
- [ ] **T054 — Verify Notes/email slice**: Test Note permission/revision/deletion, draft
  editability, transient summary text absent from AI History/audit/CRM after response, no
  Note-indexing/embedding, no save/send before approval, approved save once, rejection/expiry,
  audit, and keyboard/mobile browser journeys. **Depends on T053** (US2, AC-003–005, AC-009).

## Phase 6 — Knowledge lifecycle, RAG, and Ask CRM (S4)

- [ ] **T060 — Implement knowledge registration/lifecycle**: Register authorized non-Note source
  references, create `KnowledgeDocument` revisions, process status transitions, audit, and make
  archive/edit/delete immediately ineligible. Reject Note sources for V1 registration.
  **Depends on T003, T006, T007, T011** (FR-018–019).
- [ ] **T061 — Implement chunking and embedding pipeline**: Chunk eligible non-Note sources by
  revision, call configured embedding port, record model revision/dimension/status, and write
  idempotently; keep failed/pending/stale content unavailable. **Depends on T012, T060**
  (FR-017–019).
- [ ] **T062 — Implement pgvector retrieval RPC**: Add bounded SQL similarity function applying
  tenant, permission/source eligibility, non-Note source type, active document/chunk revision,
  ready embedding, and deletion predicates before rank/limit. Test RLS and direct RPC access.
  **Depends on T006, T061** (FR-010, FR-017–019).
- [ ] **T063 — Implement Ask CRM service**: Combine CRM Core permission-scoped search and
  authorized RAG; return citations/source scope; validate evidence-grounded answers and return
  insufficient evidence when no eligible source exists. **Depends on T040, T062, T030**
  (FR-005, FR-010).
- [ ] **T064 — Build knowledge admin and Ask CRM GenUI**: Add registration/status/revision UI,
  question entry, source citations, scope and insufficiency display, streaming/progress/cancel,
  and clear failure states. **Depends on T060–T063** (FR-024, FR-027–028).
- [ ] **T065 — Verify RAG/Ask CRM isolation and quality**: Test multiple tenants, users with
  distinct permissions, rejection of Note sources, archived/deleted/pending/failed sources,
  prompt injection fixture, unsupported question, citations, no evidence, bounded results, and
  retrieval p95. **Depends on T064** (US3, US6, AC-002, AC-006–008).

## Phase 7 — AI History, operations, and cross-cutting hardening

- [ ] **T070 — Build AI History/review UI**: Add permission-appropriate filters and detail views
  for capability, requester, provider/model, prompt version, outcome, status, review decision,
  usage, and audit references; paginate and avoid sensitive raw content. **Depends on T032–T033**.
- [ ] **T071 — Complete cross-slice audit and usage**: Verify generation, review/application,
  provider/template configuration, usage correction, cancellation, and knowledge lifecycle
  decisions each append attributable audit/usage records. **Depends on T024, T033, T043, T054,
  T065** (FR-020–022).
- [ ] **T072 — Add structured observability**: Add correlation IDs, normalized provider/retrieval
  latency/outcome metrics, usage and queue metrics, redaction, and operational alerts without raw
  prompt/output logging. **Depends on T030, T060–T063** (FR-028, NFR-011).
- [ ] **T073 — Run full accessibility/responsive/theme checks**: Audit all AI settings, GenUI,
  review, history, and knowledge surfaces for WCAG 2.2 AA-oriented keyboard semantics, focus,
  contrast, screen-reader status, 320px/mobile/tablet/desktop, and light/dark themes.
  **Depends on T023, T042, T053, T064, T070** (FR-027, AC-009).
- [ ] **T074 — Run performance/capacity verification**: Measure retrieval independently from
  provider latency; test feature latency targets and approved V1 tenant workload/quotas; record
  any justified target adjustment. **Depends on T043, T054, T065, T072** (NFRs, SC-006).
- [ ] **T075 — Validate transport reuse**: Verify Server Actions and CopilotKit adapters invoke
  the same domain services and authorization; add a test adapter representing future route/MCP
  calls without deploying REST/MCP. **Depends on T010, T031, T063** (FR-023, FR-030, AC-010).

## Phase 8 — Staging and release readiness

- [ ] **T080 — Rehearse database deployment**: Apply ordered migrations to fresh and upgraded
  disposable Supabase projects; verify extension/index/grants/RLS, backup/restore, and rollback.
  **Depends on T006, T062**.
- [ ] **T081 — Validate Vercel staging**: Deploy preview against isolated Supabase, verify
  server-only secrets, streaming/cancellation limits, auth, logs/redaction, and all slice journeys.
  **Depends on T070–T080**.
- [ ] **T082 — Pilot and rollout**: Enable by Tenant after G-1–G-4 and all isolation, audit,
  performance, accessibility, provider-health, quota, and rollback checks pass; monitor and
  document operational handoff. **Depends on T081**.
