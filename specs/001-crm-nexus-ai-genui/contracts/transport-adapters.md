# Transport Adapter Contracts

V1 exposes interactive application operations through Next.js App Router Server Actions and a
server-side CopilotKit runtime. These are adapters to the domain contracts in
`ai-domain-contracts.md`, not separate business APIs. No public REST API or MCP server is
delivered in this feature. A future adapter can expose equivalent operations by calling the
same domain services.

## Trust and response rules

1. Authenticate from the server session; derive the actor and active Tenant membership through
   the platform `AuthorizationService` on every operation.
2. Treat a URL/path/query Tenant selector as untrusted context. Revalidate it against active
   membership before reading or writing.
3. Parse `FormData`, JSON, CopilotKit tool arguments, IDs, pagination, and provider output with
   Zod. Reject unknown or malformed capability arguments.
4. Do not return an inaccessible resource's existence, contents, count, source label, or
   cross-tenant error detail.
5. Return normalized, non-sensitive errors. Never return credentials, provider request dumps,
   stack traces, raw secret references, or unauthorized source content.
6. Route/action code may invalidate Next.js UI data after successful service calls; cache keys
   include tenant, user permission scope, and source revision where caching is explicitly
   approved.

## Server Action operation map

Names below describe the conceptual operations; implementation may group exports by feature.

| Operation | Input | Domain service | Result / behavior |
|---|---|---|---|
| `requestAICapability` | Zod-validated capability, subject refs/revisions, capability input, idempotency key, optional provider selection | `AICapabilityService.execute` | `GenerationResult`, or normalized `DomainError`; draft/summary/score never mutates CRM. |
| `cancelAIGeneration` | Generation ID and idempotency key | `AICapabilityService.cancel` | Current trace state; cancellation is explicit and auditable. |
| `reviewAIProposal` | Generation ID, exact proposal revision, expected subject revisions, approve/reject, idempotency key | `ApprovalService.review` | Review state and application outcome; service reauthorizes and writes audit. |
| `listAIHistory` | Bounded page cursor, optional capability/status/date filters | `GenerationRepository.listHistory` via authorized service | Tenant/permission-filtered page; no unbounded history response. |
| `configureAIProvider` | Provider ID, display/config options, enabled state, permitted model aliases/capabilities, optional secret input handled out-of-band | Provider configuration service | Non-secret configuration response; secret values never returned or logged. |
| `publishPromptTemplate` | Stable template ID, new version content, declared inputs/output/capability | Prompt template service | New immutable published version; prior versions remain provenance-addressable. |
| `registerKnowledgeSource` | Authorized source reference and source revision | `KnowledgeIngestionService.register` | `KnowledgeStatus`; processing may be asynchronous. |
| `invalidateKnowledgeSource` | Source reference and expected revision | `KnowledgeIngestionService.invalidateSource` | Immediate retrieval ineligibility; cleanup/reindex may continue asynchronously. |

### Normalized transport error envelope

```ts
type ActionError = {
  code: string;          // stable DomainError code
  message: string;       // safe, user-presentable
  correlationId: string;
  fields?: Record<string, string>;
};
```

Server logs may include correlation ID, operation name, normalized failure category, elapsed
time, and redacted identifiers needed for diagnosis. Do not serialize untrusted prompt/result
bodies by default.

## CopilotKit runtime/GenUI contract

- The runtime receives a validated, authenticated session context and delegates each action to
  the same domain service used by Server Actions. It does not expose database access or
  provider SDK objects to the client.
- Read-only/transformative tools are limited to the approved capability list:
  `lead.summary`, `lead.score`, `notes.summarize`, `email.generate`, and `crm.ask`.
- A tool call may request a proposal or return a `GenerationResult`; it cannot execute an
  unapproved CRM mutation. Mutating tool definitions, if rendered, must stop at
  `awaiting-review`.
- Render stable UI states: `pending`, `streaming/progress`, `succeeded`, `failed`, `cancelled`,
  `awaiting-review`, `approved`, `rejected`, and `expired`.
- Source citations display only current authorized labels/references; stale or newly
  inaccessible sources are suppressed and trigger revalidation.
- Human approval is a separate user action with generation ID, proposal revision/hash, and
  expected CRM revisions. CopilotKit's local state or model-selected tool arguments are not
  proof of approval.
- On tenant switch, session expiry, capability disablement, cancellation, or permission change,
  stop/abort pending provider work where supported and reauthorize before showing or applying
  results.
- CopilotKit context and streamed results must not contain secret values. Persisted state is
  limited to canonical generation/provenance references permitted by tenant retention policy.

## Future adapter compatibility

A future route handler, REST API, or MCP tool must:

1. Authenticate its transport and resolve the same `ActorContext`.
2. Validate its external schema and translate it to the same typed domain command.
3. Call `AICapabilityService`, `ApprovalService`, `RetrievalService`, or platform/CRM services;
   never copy domain rules into the adapter.
4. Preserve normalized error, idempotency, audit, tenant isolation, and approval semantics.

MCP server deployment and public REST endpoints are explicitly outside this plan's V1 deliverable.
