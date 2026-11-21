# Quickstart: CRM Nexus AI Platform & GenUI

This is the implementation setup and verification guide for the planned feature. No application
code is implemented by this planning task. The repository currently contains specifications and
Specify tooling, not a Next.js application/package manifest.

## Prerequisites

- Node.js LTS compatible with the latest stable Next.js supported by Vercel at implementation
  time; npm is the initial package manager unless the repository establishes another standard.
- Supabase CLI and a local Docker-compatible runtime for local Postgres/Auth/Storage.
- A disposable Supabase project for staging and migration/RLS verification.
- Vercel project/environment configuration for preview and deployment.
- Git branch `001-crm-nexus-ai-genui`.
- Approval of G-1 through G-4 in `plan.md` before production provider credentials or content are
  configured.

## Planned application bootstrap

1. Create/initialize the Next.js App Router TypeScript application in the repository root;
   select strict TypeScript and pin exact dependency versions in the lockfile.
2. Add HeroUI v3 with Tailwind CSS v4 and shared light/dark theme tokens.
3. Add Supabase Auth/server integration, Zod schemas, CopilotKit React/runtime, Vitest,
   Playwright, and aimock/deterministic provider test adapters.
4. Create `supabase/migrations/` and `supabase/tests/`; enable required extensions and apply
   explicit grants plus RLS policies through versioned migrations.
5. Organize production code by the structure in `plan.md`; keep domain services independent from
   Next.js and provider SDKs.

## Local configuration

Create local ignored environment configuration from an `.env.example` containing variable names
only. Keep actual values outside version control and never include credential values in this
guide. Public Supabase URL/key are not a substitute for RLS. A Supabase service-role key, if
absolutely required for a restricted system-only task, is server-only and must not be used by
ordinary tenant requests or exposed to the browser. Provider keys are obtained at runtime via
the secret-store port; do not copy them into local test fixtures.

Configuration categories:

- Supabase project URL and client-safe key for application initialization.
- Server-side session/auth configuration.
- Server-only secret-store credentials/reference for provider adapters.
- Tenant provider/model allow-lists; never raw credentials in ordinary provider configuration.
- Versioned embedding model/revision and dimension selected before vector migration/index setup.
- Retention, usage quota, allowed source types, and ingestion execution settings.
- Test-only fake provider settings; test suite must work without real model credentials.

## Development and verification loop

1. Start the local Supabase stack and apply migrations from a clean database.
2. Seed two isolated test Tenants, multi-tenant users, roles/permissions, CRM subjects, and
   knowledge lifecycle fixtures using test-only data.
3. Run TypeScript typecheck, lint/format (according to tooling selected at bootstrap), Vitest
   unit/integration suites, and database/RLS tests.
4. Start Next.js locally and run Playwright journeys for each completed vertical slice.
5. Verify the negative cases: forged tenant selector, cross-tenant record/reference, disabled
   provider, provider timeout/invalid output, missing evidence, stale/deleted Note revision,
   rejected/expired approval, audit persistence failure, and duplicate approval.
6. Run browser checks at phone (including 320 CSS px), tablet, and desktop sizes and with both
   light and dark themes. Complete keyboard-only and screen-reader-oriented review of approval
   and citation controls.
7. Record latency separately for retrieval, provider, rendering, and history. Validate the
   feature targets and the agreed V1 workload profile before production sign-off.

## Migration and deployment verification

- Apply migrations to a fresh local/disposable project and verify forward order, constraints,
  explicit grants, RLS policy coverage, and vector extension/index readiness.
- Test with at least two Tenants and one user belonging to both. Confirm tenant switching
  invalidates old scoped reads and that no cross-tenant count, error, citation, or cache result
  leaks.
- Deploy a preview to Vercel with non-production Supabase and test-only provider adapters. Confirm
  server-only secrets are unavailable to client bundles and preview logs.
- Before production: approve secret backend, embedding model/dimension/index, retention, quotas,
  and ingestion execution; validate backup/restore and migration rollback procedures.
- Release migrations before application code that depends on them; enable AI capabilities for a
  pilot Tenant only after RLS, audit, performance, and approval tests pass.

## Completion criteria

Use the acceptance criteria and verification checklist in `plan.md` and `tasks.md`. A successful
build alone is not sufficient: cross-tenant denial, exact approval, append-only traceability,
RAG source freshness, mockable providers, accessibility, and light/dark rendering must all pass.
