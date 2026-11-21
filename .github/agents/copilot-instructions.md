# CRM Nexus Development Guidelines

Updated from the AI Platform & GenUI implementation plan on 2026-11-21.

## Active Technologies

- TypeScript with strict typechecking; pin Node.js, Next.js, and package versions at application bootstrap.
- Next.js App Router, React, HeroUI v3, Tailwind CSS v4, and CopilotKit React/runtime.
- Supabase PostgreSQL, Supabase Auth, PostgreSQL RLS, and pgvector.
- Zod for trust-boundary and persisted-data validation.
- Vitest, Playwright, and aimock/deterministic provider fakes; no live AI provider dependency in tests.
- Server-only AI provider adapters for OpenAI, Gemini, Ollama, and OpenRouter.

## Architecture and Security Rules

- Keep business logic in transport-agnostic domain services. Next.js Server Actions, Route Handlers, CopilotKit, and any future MCP adapter are thin adapters over the same services.
- Resolve user identity and active Tenant membership from authenticated server state on every operation. A caller-supplied Tenant ID is a selector, never authorization.
- Enforce tenant and permission boundaries in both domain services and PostgreSQL RLS. Set explicit database grants as well as policies. Never expose a service-role key to a browser or use it for ordinary tenant operations.
- Every tenant-owned entity and reference remains in exactly one Tenant. Exclude soft-deleted, stale, unauthorized, or otherwise unavailable CRM/knowledge content immediately.
- Keep provider credentials in a server-only secret facility; do not put secrets in domain rows, prompts, logs, audit events, errors, or client responses.
- Use a provider abstraction; no direct vendor SDK use in domain/business modules and no hidden provider fallback. Validate provider inputs/outputs with Zod.
- Use exact immutable prompt-template and embedding model revisions for provenance. RAG eligibility checks Tenant, user/source permission, active document/chunk revision, and ready embedding before vector ranking.
- AI output is a proposal, not authorization. Consequential CRM application requires explicit same-Tenant approval of the exact proposal, permission/revision revalidation, idempotency, and an append-only audit outcome.
- Treat retrieved CRM/knowledge content as untrusted evidence. Minimize persisted prompts/results and redact secrets and unnecessary personal data.
- Critical audit failure must be surfaced and must not be reported as successful application. Correct append-only history through compensating records.
- User-facing AI surfaces must be responsive, keyboard-operable, semantic, and support light and dark themes.

## Project Structure

```text
src/
  app/                   # App Router pages and transport adapters
  features/              # Vertical AI slices (lead-ai, notes-ai, ask-crm, knowledge, history)
  components/            # Shared UI, shell, and GenUI components
  domain/                # Transport-independent services, policies, and ports
  infrastructure/        # Supabase repositories, providers, embeddings, secrets
  shared/                # Zod schemas, errors, and observability
supabase/
  migrations/            # Versioned schema, grants, RLS, and RPC functions
  tests/                 # Database and tenant-isolation tests
tests/
  integration/
  e2e/
```

## Commands

The application scaffold does not yet exist. Establish and document exact package scripts
(typecheck, lint, Vitest, database/RLS tests, and Playwright) during bootstrap; do not assume
commands are available until `package.json` is created.

## Code Style

Use TypeScript strict mode, Zod at trust boundaries, explicit service/port interfaces, small
vertical feature slices, normalized error types, and tests for both allowed and denied tenant
access. Prefer server rendering; add Client Components only for interaction that requires them.

## Recent Changes

- `001-crm-nexus-ai-genui`: Planned provider-independent AI, tenant-scoped RAG/pgvector,
  CopilotKit GenUI and approval, append-only history/audit, tests, and Vercel/Supabase delivery.

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
