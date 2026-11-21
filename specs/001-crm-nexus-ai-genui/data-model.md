# Data Model: CRM Nexus AI Platform & GenUI

**Authority**: `specs/1-domain-model/spec.md` is the canonical business-domain reference used by
this plan. This document maps feature usage to that model; it does not add, rename, or redefine
domain entities.

## Conventions and invariants

- Every customer/business/configuration record below is owned by exactly one Tenant. `User` and
  the shared `Permission` vocabulary remain platform-level exceptions.
- Tenant ownership is immutable. A related tenant-owned entity must have the same Tenant ID.
- The authenticated user's active tenant membership and permissions are resolved server-side;
  transport-supplied tenant identifiers are selectors, never proof of access.
- Mutable records use logical revisions for conflict/revalidation. Soft deletion/archive is
  separate from business lifecycle and excludes records from ordinary AI context immediately.
- `AuditLog`, `AIUsageRecord`, and published prompt versions are append-only facts. Corrections
  create compensating records/events.
- Secrets are not domain attributes. Provider credentials are resolved from a server-only
  secret facility; database configuration may retain only non-secret settings/reference.
- Raw prompts/results are minimized or referenced according to the Tenant retention policy.
  Audit records contain no credentials, vectors, full Notes, or unnecessary generated content.

## Feature entities

| Entity | Ownership and principal fields | Relationships | Validation and lifecycle |
|---|---|---|---|
| **AIProvider** | Exactly one Tenant; provider identity, display name, enabled state, creation/configuring actor; optional allowed model aliases, capability set, non-secret preferences, availability/validation metadata. | May be referenced by `AIGeneration` and `AIUsageRecord`; provider adapter catalog is platform capability. | Lifecycle `enabled` / `disabled` / `retired`. Only supported provider abstractions and Tenant-enabled providers may be selected. No credential values in attributes or audit. Changes are audited. |
| **AIPromptTemplate** | Exactly one Tenant; stable template identity, purpose, version, content, lifecycle, creation/creator; optional description, declared inputs, output expectations. | A template version may be referenced by many `AIGeneration` records. | Lifecycle `draft` / `active` / `retired`. A published active version is immutable; editing publishes a new version. Inputs/output expectations are declared and validated. Generation records identify the exact version used. |
| **AIGeneration** | Exactly one Tenant and requesting User; capability/type, status, approval state, request time; optional provider/model, exact template version, same-tenant CRM subjects, minimized input/output reference, completion time, failure category, reviewer and review time. | References same-tenant Lead/Contact/Deal/Activity/Note and optional usage records; exact proposal/review outcome is recorded through this trace and append-only `AuditLog`. | Execution status: `pending`, `succeeded`, `failed`, `cancelled`; independent approval state: `not-required`, `awaiting-review`, `approved`, `rejected`, `expired`. Provider must be enabled once execution starts. Failed/cancelled generations are not approved. Consequential output remains unapplied until authorized explicit approval. For `notes.summarize`, persist only trace metadata and the source Note identity/revision; never retain the summary text. |
| **AIUsageRecord** | Exactly one Tenant; provider identity, usage time, measure/quantity, record status; optional generation/provider model, tokens, latency, request reference, estimated cost/currency, reconciliation/correction references. | Optional references to one same-Tenant `AIGeneration` and `AIProvider`. | Lifecycle `recorded` / `reconciled` / `voided`; quantities are non-negative; cost and currency co-occur. Append-only; corrections are attributable compensating records. |
| **KnowledgeDocument** | Exactly one Tenant; title, source type, content revision, status, creation/creator; optional description, source reference, source AttachmentMetadata, checksum, effective date, failure category. | Owns zero or more revision-specific `KnowledgeChunk`; optional source references must share Tenant. | Lifecycle `draft` / `processing` / `indexed` / `failed` / `archived` / `soft-deleted`. Content edits create revisions and immediately invalidate old retrieval eligibility. Only authorized, successfully indexed, current, active non-Note content is eligible in V1. Future Note-derived artifacts must retain the source Note revision and become stale when it changes or is deleted. |
| **KnowledgeChunk** | Exactly one Tenant and parent `KnowledgeDocument`; stable chunk identity, document revision, content, ordering; optional heading, offsets, provenance. | Belongs to one same-Tenant document revision and has zero or more embeddings. | Lifecycle `pending` / `indexed` / `stale` / `soft-deleted`. Chunk is eligible only while its parent document/revision is active and current. |
| **VectorEmbedding** | Exactly one Tenant and `KnowledgeChunk`; vector representation, embedding model identity/revision, creation time, status; optional dimension and indexing/failure metadata. | Belongs to one same-Tenant chunk revision; multiple representations may exist per chunk. | Lifecycle `pending` / `ready` / `stale` / `failed` / `soft-deleted`. Dimension/model metadata must match representation; only ready vectors for current eligible document/chunk revisions participate in retrieval. No vector value in audit. |
| **AuditLog** | Exactly one Tenant for tenant business events; time, action, affected entity type/identity, actor kind; optional actor ID, minimized change summary, reason, related references, correlation ID. | May refer to a tenant-owned record, including a soft-deleted record; actor authorization must be valid at event time. | Append-only while retained. Generation/provider/template/review/application/usage/knowledge transitions have attributable audit events. Critical operation is not reported successful if required audit persistence fails. |

## Existing CRM entities used as subjects and sources

AI services consume the existing canonical CRM entities through CRM Core domain-service/repository
boundaries; this plan creates none of these entities:

- **Lead**: same-tenant subject, with canonical status/source/owner/revision. Lead Summary and
  Lead Scoring require subject-read permission; scoring is explanatory and never changes Lead
  fields automatically.
- **Contact** and **Deal**: eligible same-tenant subject references for Email Generation;
  permissions and source fields follow CRM Core. Email output is a draft, not a sent message.
- **Activity**: may be an authorized related source for Lead Summary or a subject reference only
  where the capability contract permits it.
- **Note**: may be used only by the dedicated transient on-demand Notes Summarization capability,
  and only for an authorized current revision. Return the summary only in the request response;
  persist trace metadata and the exact source revision, but not summary text. Do not index, embed,
  or use Notes for Ask CRM or other V1 AI capabilities. Future-derived artifacts must retain the
  source revision and become stale on change or deletion.
- **AttachmentMetadata**: may identify an approved, same-tenant knowledge source; quarantined or
  unavailable content is excluded.

CRM Core lifecycles, explicit Lead conversion, one primary Contact per Deal, single-parent Note,
authorship, revision history, soft deletion, and same-tenant relationships remain authoritative.

## Relationships

```text
Tenant
 ├── AIProvider
 ├── AIPromptTemplate (stable identity → immutable versions)
 ├── AIGeneration ── requester: User (active membership in Tenant)
 │    ├── exact AIPromptTemplate version (optional)
 │    ├── AIProvider/model used (once execution begins)
 │    ├── zero or more AIUsageRecord
 │    └── zero or more same-tenant CRM subjects
 ├── AIUsageRecord
 ├── KnowledgeDocument (source revision)
 │    └── KnowledgeChunk (same document revision)
 │         └── VectorEmbedding (model/revision/dimension)
 └── AuditLog (append-only decisions and transitions)
```

Cross-tenant joins/references must be rejected at both service and database boundaries.
Composite `(tenant_id, id)` foreign keys or equivalent database constraints should enforce
same-tenant parent references where relationally applicable. RLS must protect each exposed table,
including history, provider configuration, prompts, knowledge chunks, and vectors.

## State and transition rules

### Generation and review

1. A validated, authorized request creates a trace in `pending` with approval `not-required` or
   `awaiting-review` as determined by the capability's declared side effect.
2. Provider success yields a validated result and `succeeded`; provider/validation failure
   yields `failed`; user/system stop yields `cancelled`. Failure and cancellation never carry an
   approved result.
3. A consequential proposal remains `awaiting-review` and unapplied. A reviewer decision records
   actor/time and exact proposal revision in generation provenance/audit. State becomes
   `approved`, `rejected`, or `expired`.
4. An approved application revalidates active tenant membership, permission, proposal identity,
   subject identity/revision, and idempotency before calling the existing CRM domain command.
   Record application outcome; rejection, expiry, failure, or cancellation cannot be retried as
   an approval.

### Knowledge

1. Registration/revision creates `KnowledgeDocument` in `draft` or `processing`; ingestion
   creates pending chunk and embedding records for that exact content revision.
2. Only a successful current revision advances to `indexed`; chunk lifecycle is `indexed` and
   embedding lifecycle `ready`.
3. Source change/archive/deletion immediately makes prior document/chunk/embedding revisions
   ineligible (`stale`/archived/deleted as appropriate), before asynchronous reindex or cleanup.
4. Retrieval requires same-tenant source ownership and current revision, active source state,
   ready embedding, source permission, and authorized requesting user. Similarity ranking and
   result limiting occur only after those constraints.

## Storage notes (implementation detail, not domain changes)

- Use PostgreSQL migrations with RLS and explicit grants; index common `(tenant_id, status,
  created_at)` and foreign-key lookup paths based on query plans.
- Use pgvector with a dimension pinned per embedding index/model revision. Similarity uses a
  server-side SQL function/RPC with a fixed result contract and bounded `match_count`.
- Apply critical domain mutation and required audit in one transaction/RPC boundary where
  feasible; otherwise fail closed and do not report the mutation as complete.
- Store optional large/source payloads only by protected reference and according to retention
  policy; keep provider secrets out of ordinary rows.
- AI history list/detail reads are permission-filtered and paginated; latest 100 are the
  performance target, not a reason to return unbounded output.

## Model-change review

No canonical domain change is proposed. If implementation requires a new persistent business
concept, ownership rule, relationship, lifecycle, or retention exception, amend and approve the
canonical domain model and dependent specs before schema implementation. Internal tables,
indexes, outbox records, secret references, and provider-specific metadata must not become
undocumented business entities.
