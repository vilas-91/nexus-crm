# Research: CRM Nexus AI Platform & GenUI

**Date**: 2026-11-21  
**Scope**: Resolve technical design choices for the approved AI Platform & GenUI specification.
All links below are official framework/vendor documentation; no private repository content was
sent to external services.

## Decisions

### 1. Next.js App Router transports are adapters, not the domain layer

- **Decision**: Use Server Components for read-oriented server rendering where useful and
  focused Client Components for CopilotKit/HeroUI interaction. Server Actions, Route Handlers,
  and the CopilotKit runtime validate requests, resolve authenticated actor context, call
  transport-agnostic services, shape responses, and perform UI revalidation only.
- **Rationale**: Next.js Server Functions are invoked through network requests (POST); they are
  reachable independently of the visible UI. Authentication and authorization therefore must
  be checked inside every action, not assumed from the page or button that invoked it. Reusing
  domain services also satisfies the required future REST/MCP boundary.
- **Alternatives considered**: Put domain operations directly in actions (rejected: duplicates
  business rules and impedes future transport reuse); make all interactions public HTTP APIs
  (rejected: not required for V1 and expands surface area).
- **Source**: [Next.js — Mutating Data](https://nextjs.org/docs/app/getting-started/mutating-data),
  [Next.js — Server Actions and Mutations](https://nextjs.org/docs/app/guides/server-actions),
  [Next.js — Data Security](https://nextjs.org/docs/app/guides/data-security).

### 2. PostgreSQL RLS and grants form the database enforcement layer

- **Decision**: Enable RLS on every exposed tenant-owned table, set explicit grants as well as
  policies, and test both. Build row policies from authenticated identity and active tenant
  membership; reinforce them with service-level permission/revision checks. Keep
  `service_role` server-only and out of normal user operations.
- **Rationale**: Policies and table privileges are separate checks; policies do not revoke
  grants. Supabase documents that `service_role` bypasses RLS. RLS is defense in depth and also
  protects access paths beyond the Next.js UI.
- **Alternatives considered**: Application-only tenant filters (rejected: one omitted filter
  would disclose data); RLS-only authorization (rejected: domain permission and review rules
  still belong in services); `service_role` for convenience (rejected: bypasses the intended
  database boundary).
- **Source**: [Supabase — Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security),
  [Supabase — Securing your API](https://supabase.com/docs/guides/api/securing-your-api).

### 3. Supabase PostgreSQL pgvector with a constrained server-side retrieval function

- **Decision**: Store canonical knowledge metadata and vector representations in Supabase
  PostgreSQL using pgvector. Pin vector dimensions to the deployment's selected embedding model
  and revision. Call a narrowly scoped Postgres function/RPC that applies tenant, permission,
  document/chunk state, revision, and deletion eligibility before ordering and limiting by
  similarity.
- **Rationale**: Supabase's vector guide documents the `vector` extension/type and distance
  operators. It notes PostgREST does not expose pgvector similarity operators directly, so a
  Postgres function called through RPC is an appropriate application boundary. Eligibility
  predicates must be applied before ranking; filtering a global result set after retrieval is
  unsafe and can leak existence/count information.
- **Alternatives considered**: Separate vector database (rejected for V1 because Supabase
  pgvector is the constitutional default and adds an additional tenant boundary); client-side
  vector filtering (rejected: unauthorized data already crossed the boundary); one mutable
  embedding column with no model revision (rejected: dimensions and provenance can change).
- **Source**: [Supabase — Vector columns](https://supabase.com/docs/guides/ai/vector-columns),
  [Supabase — pgvector](https://supabase.com/docs/guides/ai).

### 4. CopilotKit presents interaction state; application services own approval

- **Decision**: Use CopilotKit for streaming/progress and typed tool/proposal rendering, with
  explicit waiting-for-review, approved, rejected, expired, failed, and cancelled presentation.
  Any actual CRM application must invoke the ordinary approval service with actor, exact
  generation/proposal revision, and current CRM revision. CopilotKit state or a model tool call
  is never authorization.
- **Rationale**: CopilotKit documents human-in-the-loop patterns for gathering confirmation and
  user choice. The feature's stronger domain invariant requires approval to be explicit,
  attributable, same-tenant, revalidated, and durable regardless of the UI technology.
- **Alternatives considered**: Let the model execute a mutation after tool selection (rejected:
  violates the human approval/domain invariant); implement approval only as a client-side dialog
  (rejected: bypassable and not auditable); omit GenUI state (rejected: fails the requested
  interaction and progress model).
- **Source**: [CopilotKit — Human-in-the-loop overview](https://docs.copilotkit.ai/langgraph-python/human-in-the-loop).

### 5. HeroUI v3 with Tailwind CSS v4 and accessible theme tokens

- **Decision**: Adopt HeroUI v3 components with Tailwind CSS v4; centralize light/dark theme
  tokens and compose reusable GenUI, approval, history, and status components. Verify keyboard,
  screen-reader semantics, focus, contrast, and responsive layout for the composed experience.
- **Rationale**: HeroUI's official documentation describes a React component library built on
  Tailwind CSS v4 and React Aria Components, with built-in typing, keyboard navigation, focus
  management, and screen-reader support. Components aid accessibility but do not replace
  feature-level checks.
- **Alternatives considered**: A bespoke design system (rejected: more implementation and
  accessibility maintenance); relying on component defaults without page-level testing
  (rejected: CopilotKit and approval flows combine multiple interactive components).
- **Source**: [HeroUI React — Introduction](https://heroui.com/docs/react/getting-started).

### 6. Provider, embedding, and secret access are independent server-side ports

- **Decision**: Define provider execution, embedding generation, and secret resolution as
  separate server-only ports. Normalize provider outputs/errors and validate them with Zod.
  Keep adapter-specific SDK calls, model names, and credential values out of domain modules.
- **Rationale**: This preserves the required OpenAI/Gemini/Ollama/OpenRouter replaceability and
  allows a tenant-selected provider without coupling CRM rules to a vendor. Embeddings can have
  different availability, dimensions, and lifecycle from chat completion.
- **Alternatives considered**: Direct calls to individual SDKs in each capability (rejected:
  vendor lock-in and duplicated errors); using one provider SDK interface for embeddings and
  chat without version metadata (rejected: hides materially different model/dimension behavior);
  storing provider credentials in `AIProvider` attributes or audit (rejected: prohibited by
  the specification).
- **Sources**: Feature requirements FR-003, FR-015–016, FR-021–023; canonical domain model
  `AIProvider` and `VectorEmbedding` constraints; [Supabase — Vault](https://supabase.com/docs/guides/database/vault)
  (one possible managed secret facility; deployment selection remains an implementation gate).

### 7. Testing strategy follows trust boundaries and vertical slices

- **Decision**: Use Vitest for domain/service tests; database migration and RLS tests for
  tenant enforcement; aimock or deterministic fakes for provider contracts; Playwright for
  complete interactive slices and responsive/accessibility behavior. Run all without live
  provider credentials.
- **Rationale**: Tenant isolation, approval, provider mocking, RAG eligibility, and audit
  failures are correctness/security properties that span boundaries and need direct negative
  tests. Feature-spec NFRs explicitly identify Vitest, Playwright, and aimock.
- **Alternatives considered**: Live-provider-only acceptance testing (rejected: unreliable,
  costly, and can disclose test content); unit tests only (rejected: cannot establish RLS,
  server-action, browser, or responsive behavior); browser tests only (rejected: slow and weak
  at isolating business invariants).
- **Source**: CRM Nexus Constitution, Principle XI and AI spec FR-025/AC-008.

## Resolved planning unknowns and deferred deployment inputs

The approved feature spec has no open product questions. This plan resolves the architecture
choices while preserving the explicit deployment-policy inputs:

| Input | Resolved plan decision | Required before |
|---|---|---|
| Provider credential handling | SecretStore port; keep secret values outside ordinary domain records/logs. Use an approved managed secret facility. | Persisting any tenant credential |
| Embedding model/dimension | Versioned embedding adapter and metadata; pin one V1 dimension/model per vector index; no reinterpretation on change. | Creating the production vector migration/index |
| Prompt/output retention | Minimize persisted content and retain exact provenance; retention periods remain tenant/deployment policy. | Accepting production data |
| AI volume/concurrency/quota | Instrument provider, token/usage, latency, retrieval, and queue; benchmark an agreed workload profile. | Production capacity sign-off |
| Ingestion execution | Idempotent, tenant/revision-scoped async execution selected from the deployed platform's supported facilities. | Enabling production ingestion |

These are not requirement ambiguities; changing them does not change canonical ownership or
service contracts.
