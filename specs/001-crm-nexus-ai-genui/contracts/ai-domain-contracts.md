# AI Domain Service Contracts

These are planning contracts for TypeScript/Zod interfaces. They describe the service boundary,
not concrete route schemas, database rows, or provider SDK types. All commands are validated at
entry and results at exit. `ActorContext` is resolved from the authenticated server session and
active membership; callers cannot manufacture it by passing a Tenant ID.

## Shared types

```ts
type TenantId = string;
type UserId = string;
type RecordRef = {
  type: "lead" | "contact" | "deal" | "activity" | "note";
  id: string;
  revision: number;
};

type ActorContext = {
  userId: UserId;
  tenantId: TenantId;
  membershipRevision: number;
  correlationId: string;
};

type SourceCitation = {
  sourceType: "crm_record" | "knowledge_document";
  sourceId: string;
  sourceRevision: number;
  label: string;
  excerpt?: string; // minimized and permission-filtered
};

type DomainError =
  | { code: "UNAUTHENTICATED" | "TENANT_ACCESS_DENIED" | "PERMISSION_DENIED" }
  | { code: "NOT_FOUND" } // do not reveal inaccessible cross-tenant existence
  | { code: "VALIDATION_FAILED"; fields?: Record<string, string> }
  | { code: "CONFLICT"; currentRevision?: number }
  | { code: "PROVIDER_UNAVAILABLE" | "PROVIDER_TIMEOUT" | "PROVIDER_INVALID_OUTPUT" }
  | { code: "INSUFFICIENT_EVIDENCE" }
  | { code: "CANCELLED" | "EXPIRED" | "ALREADY_APPLIED" }
  | { code: "AUDIT_WRITE_FAILED" | "PERSISTENCE_FAILED" };
```

In implementation, use branded IDs and Zod schemas rather than accepting unvalidated strings.
Error payloads must not include provider secrets, raw payload dumps, or unauthorized resource
details.

## Capability contract

```ts
type Capability =
  | "lead.summary"
  | "lead.score"
  | "notes.summarize"
  | "email.generate"
  | "crm.ask";

type CapabilityRequest = {
  capability: Capability;
  subjectRefs: RecordRef[];
  input: unknown;          // capability-specific Zod schema
  idempotencyKey: string;
  requestedProviderId?: string; // selection hint; must be enabled/authorized
};

type GenerationResult = {
  generationId: string;
  capability: Capability;
  status: "pending" | "succeeded" | "failed" | "cancelled";
  approvalState:
    | "not-required"
    | "awaiting-review"
    | "approved"
    | "rejected"
    | "expired";
  result?: unknown;         // capability-specific Zod schema
  citations: SourceCitation[];
  createdAt: string;
  completedAt?: string;
};

interface AICapabilityService {
  execute(actor: ActorContext, request: CapabilityRequest): Promise<GenerationResult>;
  cancel(actor: ActorContext, generationId: string): Promise<GenerationResult>;
  get(actor: ActorContext, generationId: string): Promise<GenerationResult>;
}
```

Capability-specific output schemas:

- `lead.summary`: concise summary plus citations, no write command.
- `lead.score`: bounded score/label and validated explanatory factors, no CRM-field mutation.
- `notes.summarize`: transient on-demand summary and exact Note revision citation; only the
  accessible current revision. Return summary text only in the request response; persist trace
  metadata and source revision only. Do not retain summary text, index/embed Note content, or use
  it for retrieval or another V1 AI capability. Reject stale or deleted revisions.
- `email.generate`: editable draft, recipients/subject/body as proposal content only; no external
  send in V1 and no automatic CRM persistence.
- `crm.ask`: answer, citations, declared evidence scope, and an insufficient-evidence outcome.
  No unsupported assertion may be disguised as an evidence-backed answer.

## Authorization and CRM ports

```ts
interface AuthorizationService {
  resolveActor(session: AuthenticatedSession, requestedTenant?: TenantId): Promise<ActorContext>;
  assertPermission(actor: ActorContext, permissionCode: string): Promise<void>;
  assertRecordAccess(actor: ActorContext, ref: RecordRef, action: "read" | "write"): Promise<void>;
}

interface CRMContextPort {
  loadPermittedContext(
    actor: ActorContext,
    refs: RecordRef[],
    capability: Capability,
  ): Promise<{ records: unknown[]; citations: SourceCitation[] }>;

  applyApprovedCommand(
    actor: ActorContext,
    command: ApprovedCRMCommand,
    expectedRevisions: Record<string, number>,
    idempotencyKey: string,
  ): Promise<{ outcome: "applied"; updatedRefs: RecordRef[] }>;
}
```

The existing CRM Core domain service owns lifecycle and mutation rules. AI services must not
write directly to CRM tables or create a parallel Lead/Contact/Deal command model.

## Provider and embedding ports

```ts
type ProviderRequest = {
  capability: Capability;
  modelAlias: string;
  promptVersion?: string;
  systemInstructions: string;
  untrustedContext: Array<{ text: string; citation: SourceCitation }>;
  input: unknown;
  signal?: AbortSignal;
};

type ProviderResponse = {
  output: unknown;
  providerId: string;
  normalizedModel: string;
  usage?: { inputTokens?: number; outputTokens?: number; totalTokens?: number };
  latencyMs: number;
  providerRequestRef?: string;
};

interface AIProvider {
  execute(request: ProviderRequest): Promise<ProviderResponse>;
  healthCheck(): Promise<{ available: boolean; checkedAt: string }>;
}

interface ProviderRegistry {
  getEnabled(
    actor: ActorContext,
    providerId: string,
    capability: Capability,
  ): Promise<AIProvider>;
}

interface EmbeddingProvider {
  embed(input: string[], modelRevision: string): Promise<{
    vectors: number[][];
    modelIdentity: string;
    modelRevision: string;
    dimensions: number;
  }>;
}
```

Adapters cover OpenAI, Gemini, Ollama, and OpenRouter. They normalize errors, usage, model
aliases, cancellation, and streaming. No adapter may choose an alternate provider silently.
Provider responses are schema-validated before persistence or presentation. Model revisions and
dimensions must match the embedding index contract.

## Prompt, secret, persistence, and audit ports

```ts
interface PromptTemplateService {
  resolveActive(actor: ActorContext, capability: Capability): Promise<{
    templateId: string;
    version: string;
    instructions: string;
    declaredInputs: string[];
    outputSchemaId: string;
  }>;
}

interface SecretStore {
  resolve(reference: string, tenantId: TenantId): Promise<string>;
}

interface GenerationRepository {
  createPending(actor: ActorContext, request: ValidatedCapabilityRequest): Promise<string>;
  appendOutcome(actor: ActorContext, outcome: ValidatedGenerationOutcome): Promise<void>;
  appendUsage(actor: ActorContext, usage: ValidatedUsage): Promise<void>;
  listHistory(actor: ActorContext, query: ValidatedHistoryQuery): Promise<GenerationPage>;
}

interface AuditService {
  append(actor: ActorContext, event: MinimizedAuditEvent): Promise<void>;
}
```

`SecretStore.resolve` is server-only and returns a secret solely to the adapter invocation. The
reference/value must never be serialized into a client response, prompt, database audit payload,
error, or ordinary application log. Required audit writes are part of the critical operation's
durable outcome.

## Review/application contract

```ts
type ReviewCommand = {
  generationId: string;
  proposalRevision: string; // immutable version or content hash
  expectedSubjectRevisions: Record<string, number>;
  decision: "approve" | "reject";
  idempotencyKey: string;
};

interface ApprovalService {
  review(actor: ActorContext, command: ReviewCommand): Promise<{
    approvalState: "approved" | "rejected" | "expired";
    applicationOutcome?: "applied" | "not-applied" | "conflict";
    updatedRefs?: RecordRef[];
  }>;
}
```

`review` verifies reviewer permission and same-Tenant membership, current generation state,
exact proposal identity, subject revisions, and one-time idempotency. An approved decision is
durably audited before success is returned. `reject` never applies the proposal. A revision
conflict or stale permission returns a conflict/denial and leaves CRM unchanged.

## Retrieval contract

```ts
interface KnowledgeIngestionService {
  register(actor: ActorContext, source: AuthorizedKnowledgeSource): Promise<KnowledgeStatus>;
  reindex(actor: ActorContext, documentId: string, expectedRevision: number): Promise<void>;
  invalidateSource(actor: ActorContext, sourceRef: RecordRef): Promise<void>;
}

interface RetrievalService {
  retrieve(actor: ActorContext, request: {
    query: string;
    embedding: number[];
    expectedModelRevision: string;
    limit: number;
    sourceScope?: string[];
  }): Promise<Array<{
    chunkId: string;
    documentId: string;
    documentRevision: number;
    text: string;
    similarity: number;
    citation: SourceCitation;
  }>>;
}
```

Retrieval is performed by a database RPC whose eligibility predicates enforce tenant, current
document/chunk revision, `indexed`/`ready` state, non-deletion/archive, source permission, and
bounded result count before similarity ranking. The service revalidates citations against
current source revisions before composing the prompt.
