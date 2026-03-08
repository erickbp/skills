# Multi-Tenancy Migration: Task Decomposition

## Overview

Migration of a SaaS analytics platform from single-tenant (separate deployments per customer on subdomains) to a shared-infrastructure multi-tenant architecture. All tenants will share the same database, application instances, Redis, and Celery workers, with logical data isolation.

**Stack:** Next.js (frontend), FastAPI (backend), PostgreSQL (database), Redis (caching), Celery (async jobs)
**Scale:** 40 tables, ~200 API endpoints, 15 background job types, 23 current tenants targeting 500+

---

## Task 1: Design and Implement the Tenant Model and Registry

### Goal
Create the foundational `tenants` table and Tenant model that serves as the registry for all tenants in the system. This is the root entity that every other piece of tenant-scoped data will reference.

### Scope
- Create an Alembic migration adding a `tenants` table with columns: `id` (UUID primary key), `slug` (unique, URL-safe identifier matching the subdomain), `name`, `domain` (custom domain, nullable), `plan` (enum or string), `status` (active/suspended/deleted), `settings` (JSONB for tenant-specific config), `created_at`, `updated_at`.
- Create the corresponding SQLAlchemy model (`Tenant`).
- Add a unique index on `slug` and on `domain`.
- Seed the table with the 23 existing customer records (slugs derived from current subdomains).
- Add CRUD API endpoints: `GET /admin/tenants`, `POST /admin/tenants`, `GET /admin/tenants/{id}`, `PATCH /admin/tenants/{id}`, `DELETE /admin/tenants/{id}` (soft delete).
- Protect admin endpoints with a superadmin authorization check.

### Dependencies
None -- this is the foundational task.

### Acceptance Criteria
- The `tenants` table exists and the Alembic migration runs cleanly (upgrade and downgrade).
- All 23 existing customers have a row in the tenants table after seed migration.
- Admin CRUD endpoints function correctly and are protected.
- The `Tenant` SQLAlchemy model is importable and usable across the codebase.

### Verification Steps
1. Run `alembic upgrade head` and confirm no errors.
2. Run `alembic downgrade -1` and confirm clean rollback, then re-upgrade.
3. Query `SELECT count(*) FROM tenants;` and confirm 23 rows.
4. Write and run pytest tests for each admin endpoint (create, read, update, soft-delete).
5. Confirm unauthorized requests to admin endpoints return 403.

---

## Task 2: Add `tenant_id` Foreign Key to All Existing Tables

### Goal
Add a `tenant_id` column (UUID, foreign key to `tenants.id`) to every existing data table so that all rows are associated with a specific tenant.

### Scope
- Create an Alembic migration that adds a `tenant_id UUID NOT NULL` column (with a default pointing to a migration-time constant for backfill) and a foreign key constraint to each of the 40 existing tables.
- Add a composite index on `(tenant_id, <existing_primary_key>)` for each table.
- Replace any existing unique constraints that should now be unique-per-tenant with composite unique constraints that include `tenant_id` (e.g., a user's email should be unique per tenant, not globally).
- Update all SQLAlchemy models to include the `tenant_id` column and the foreign key relationship.
- Backfill existing data: assign every existing row the correct `tenant_id` based on the deployment it came from (this requires a data mapping from the 23 single-tenant deployments).

### Dependencies
- Task 1 (Tenant model and table must exist).

### Acceptance Criteria
- All 40 tables have a non-nullable `tenant_id` column with a foreign key to `tenants.id`.
- All existing data rows have the correct `tenant_id` populated.
- Unique constraints that should be tenant-scoped are updated to composite constraints including `tenant_id`.
- All SQLAlchemy models reflect the new column.

### Verification Steps
1. Run `alembic upgrade head` successfully.
2. Run `SELECT table_name FROM information_schema.columns WHERE column_name = 'tenant_id';` and confirm all 40 tables are listed.
3. Verify foreign key constraints exist: query `information_schema.table_constraints`.
4. Spot-check 5 tables to confirm no rows have a NULL `tenant_id`.
5. Run the existing test suite and note/triage any failures (expected at this stage since query logic has not been updated yet).

---

## Task 3: Implement Tenant Resolution Middleware

### Goal
Build the middleware/dependency that identifies which tenant a request belongs to, based on the subdomain (or a header for API clients), and makes the tenant context available throughout the request lifecycle.

### Scope
- Create a FastAPI middleware or dependency (`get_current_tenant`) that:
  - Extracts the subdomain from the `Host` header (e.g., `acme.analytics.io` -> slug `acme`).
  - Alternatively accepts an `X-Tenant-ID` header (for service-to-service calls).
  - Looks up the tenant in the database (with Redis caching -- TTL 5 minutes).
  - Returns 404 if tenant not found, 403 if tenant is suspended.
  - Stores the resolved `Tenant` object in a request-scoped context variable (Python `contextvars`).
- Create a `get_tenant_id()` utility function that retrieves the current tenant ID from context, usable in non-request contexts (e.g., Celery tasks).
- Create a `TenantContext` context manager for explicitly setting tenant context in background jobs and scripts.

### Dependencies
- Task 1 (Tenant model must exist).

### Acceptance Criteria
- Requests to `acme.analytics.io/api/...` resolve to the `acme` tenant.
- Requests to unknown subdomains return 404.
- Requests to suspended tenants return 403.
- Tenant lookups are cached in Redis (verified by checking Redis keys).
- `get_tenant_id()` returns the correct tenant ID within a request.
- `TenantContext` allows setting tenant context explicitly in non-request code.

### Verification Steps
1. Write pytest tests with mocked `Host` headers for valid, invalid, and suspended tenants.
2. Test `X-Tenant-ID` header resolution.
3. Verify Redis cache hit/miss behavior with mock Redis or test Redis instance.
4. Test `get_tenant_id()` returns correct value in request context.
5. Test `TenantContext` context manager sets and clears context correctly.

---

## Task 4: Implement Row-Level Tenant Filtering in the ORM Layer

### Goal
Ensure every database query automatically filters by the current tenant, preventing cross-tenant data leakage without requiring manual `WHERE tenant_id = ...` clauses in every query.

### Scope
- Create a custom SQLAlchemy session class or event listener that automatically appends `.filter(Model.tenant_id == current_tenant_id)` to all SELECT queries for tenant-scoped models.
- Create a `TenantScopedQuery` mixin or base class that tenant-scoped models inherit from.
- Override the `Session.add()` / `Session.bulk_save_mappings()` methods (or use `before_insert` events) to automatically set `tenant_id` on new records.
- Create a `bypass_tenant_filter()` context manager for admin/superadmin operations that need cross-tenant access.
- Add a SQLAlchemy event listener on `before_update` and `before_delete` to verify the `tenant_id` matches the current context (prevent accidental cross-tenant writes).

### Dependencies
- Task 2 (`tenant_id` column must exist on all models).
- Task 3 (Tenant context must be resolvable).

### Acceptance Criteria
- A `SELECT * FROM dashboards` query in the context of tenant A returns only tenant A's dashboards, without explicit filtering in application code.
- Inserting a new record automatically sets `tenant_id` to the current tenant.
- Attempting to update or delete a record belonging to a different tenant raises an exception.
- `bypass_tenant_filter()` correctly disables filtering for admin operations.
- No query in the codebase can accidentally return cross-tenant data.

### Verification Steps
1. Write pytest tests that create data for two tenants and verify queries in tenant A's context only return tenant A's data.
2. Test that inserting a record without explicit `tenant_id` auto-populates it.
3. Test that cross-tenant update/delete raises an error.
4. Test that `bypass_tenant_filter()` returns data from all tenants.
5. Run `EXPLAIN` on representative queries and confirm `tenant_id` filter is present in the query plan.

---

## Task 5: Update All API Endpoints to Be Tenant-Aware

### Goal
Modify all ~200 API endpoints to use the tenant-scoped database session and tenant context, ensuring every endpoint only accesses data belonging to the resolved tenant.

### Scope
- Inject the `get_current_tenant` dependency into all API route handlers (or apply it globally via a router-level dependency).
- Update all endpoint handler functions to use the tenant-scoped session from Task 4 instead of the unscoped session.
- Audit each endpoint for any raw SQL queries and update them to include `tenant_id` filtering.
- Update any endpoint that references tenant-specific configuration (e.g., branding, feature flags) to pull from the `Tenant.settings` JSONB.
- Handle endpoints that are intentionally cross-tenant (admin endpoints) by using the `bypass_tenant_filter()` context manager.
- Group the 200 endpoints into logical batches (e.g., by router/module) for incremental migration.

### Dependencies
- Task 3 (Tenant resolution middleware).
- Task 4 (Row-level tenant filtering).

### Acceptance Criteria
- Every API endpoint, when called with a tenant-specific subdomain, only returns/modifies data belonging to that tenant.
- No endpoint returns data from another tenant (verified by cross-tenant access tests).
- Admin endpoints still function correctly with cross-tenant access where appropriate.
- All existing API tests pass with tenant context properly set up in test fixtures.

### Verification Steps
1. Update the pytest test fixtures to set up tenant context (mock `Host` header or use `TenantContext`).
2. For each endpoint module, run existing tests and confirm they pass.
3. Write at least one cross-tenant isolation test per router module: create data under tenant A, call the endpoint under tenant B, and confirm the data is not returned.
4. Run a full API test suite pass and confirm zero regressions.
5. Manually test 5 critical endpoints (e.g., dashboard list, report generation, user profile) via curl with different `Host` headers.

---

## Task 6: Update Redis Caching to Be Tenant-Scoped

### Goal
Ensure all Redis cache keys are namespaced by tenant to prevent cross-tenant cache pollution.

### Scope
- Audit all Redis key patterns used in the application.
- Create a `tenant_cache_key(key: str) -> str` utility that prepends `tenant:{tenant_id}:` to all cache keys.
- Update all cache read/write operations to use the tenant-scoped key utility.
- Update cache invalidation logic to scope invalidation to the current tenant.
- Add a `flush_tenant_cache(tenant_id)` admin utility to clear all cached data for a specific tenant (e.g., on plan change or data reset).
- Ensure the tenant resolution cache from Task 3 uses a global (non-tenant-scoped) namespace, since it is looked up before tenant context is established.

### Dependencies
- Task 3 (Tenant context must be available to build scoped keys).

### Acceptance Criteria
- All Redis keys for tenant-scoped data include the tenant ID in their key structure.
- Cache hits for tenant A never return data cached for tenant B.
- `flush_tenant_cache()` removes all keys for a specific tenant without affecting others.
- The tenant resolution cache itself remains globally accessible.

### Verification Steps
1. Run the application with two test tenants and inspect Redis keys (via `redis-cli KEYS *`) to confirm proper namespacing.
2. Write pytest tests that cache data under tenant A, switch to tenant B, and confirm a cache miss.
3. Test `flush_tenant_cache()` clears only the target tenant's keys.
4. Verify the tenant resolution cache is not affected by tenant-scoped flush.

---

## Task 7: Update Celery Background Jobs to Be Tenant-Aware

### Goal
Ensure all 15 background job types correctly propagate tenant context, execute in the scope of the correct tenant, and do not leak data across tenants.

### Scope
- Create a custom Celery base task class (`TenantTask`) that:
  - Accepts `tenant_id` as a required argument (or extracts it from task headers).
  - Sets up the `TenantContext` (from Task 3) in the `before_start` hook.
  - Tears down the tenant context in the `after_return` hook.
- Update all 15 task definitions to inherit from `TenantTask`.
- Update all call sites that enqueue tasks (`.delay()`, `.apply_async()`) to pass the current `tenant_id`.
- Add tenant-scoped logging: include `tenant_id` in all Celery log records.
- Add a Celery task router or queue-per-tenant strategy if needed for tenant-level rate limiting (evaluate and decide; at minimum, document the decision).

### Dependencies
- Task 3 (Tenant context / `TenantContext` context manager).
- Task 4 (Tenant-scoped DB queries must work within task context).

### Acceptance Criteria
- Every Celery task runs with the correct tenant context.
- Database queries within tasks are automatically tenant-scoped.
- Tasks cannot accidentally process data belonging to a different tenant.
- Task logs include `tenant_id` for observability.
- Enqueueing a task without a `tenant_id` raises a validation error.

### Verification Steps
1. Write pytest tests for each of the 15 task types with mocked tenant context, verifying they process only the correct tenant's data.
2. Test that enqueueing a task without `tenant_id` raises an error.
3. Inspect Celery worker logs and confirm `tenant_id` appears in log entries.
4. Run a task for tenant A and verify it does not touch tenant B's data (integration test with test database).

---

## Task 8: Data Migration -- Consolidate 23 Single-Tenant Databases into One

### Goal
Migrate all data from the 23 existing single-tenant PostgreSQL databases into the single shared multi-tenant database, assigning correct `tenant_id` values.

### Scope
- Write a migration script (Python, using SQLAlchemy) that:
  - Connects to each of the 23 source databases.
  - Reads all rows from each table.
  - Assigns the correct `tenant_id` (looked up from the tenants registry).
  - Handles primary key conflicts (re-maps UUIDs or auto-increment IDs if necessary).
  - Re-maps foreign key references to match new primary keys.
  - Inserts data into the target multi-tenant database.
- Handle data ordering to respect foreign key constraints (topological sort of tables).
- Generate a migration report: row counts per table per tenant, before and after.
- Build in idempotency: the script should be re-runnable without creating duplicates.
- Create a rollback plan: export each tenant's data as a separate backup before migration.

### Dependencies
- Task 1 (Tenant registry must exist with all 23 tenants).
- Task 2 (`tenant_id` columns must exist on all tables).

### Acceptance Criteria
- All data from all 23 databases is present in the shared database.
- Every row has the correct `tenant_id`.
- Foreign key relationships are preserved (no orphaned references).
- Row counts match between source and target (per table, per tenant).
- The migration is idempotent -- running it twice does not create duplicates.

### Verification Steps
1. Run the migration script against a staging environment with a copy of production data.
2. Compare row counts: `SELECT tenant_id, count(*) FROM <table> GROUP BY tenant_id` for all 40 tables.
3. Run foreign key integrity checks: `SELECT * FROM <child_table> WHERE <fk_column> NOT IN (SELECT id FROM <parent_table>)` for all FK relationships.
4. Spot-check 10 records per tenant (across different tables) to verify data integrity.
5. Run the migration script a second time and confirm zero new rows inserted.

---

## Task 9: Update Authentication and Authorization for Multi-Tenancy

### Goal
Modify the authentication system so that users are scoped to tenants, can belong to multiple tenants, and JWTs/sessions include tenant context.

### Scope
- Create a `tenant_users` junction table (if users can belong to multiple tenants) with columns: `user_id`, `tenant_id`, `role` (owner/admin/member/viewer), `invited_at`, `joined_at`.
- Update the login flow to include tenant context: after authentication, if a user belongs to multiple tenants, either auto-select based on subdomain or prompt for tenant selection.
- Update JWT token payload to include `tenant_id` and `tenant_role`.
- Update the `get_current_user` dependency to validate that the authenticated user belongs to the tenant resolved from the subdomain.
- Return 403 if a user tries to access a tenant they do not belong to.
- Update user invitation and signup flows to be tenant-scoped.
- Handle the superadmin role: users with a global admin flag can access any tenant.

### Dependencies
- Task 1 (Tenant model).
- Task 3 (Tenant resolution middleware).

### Acceptance Criteria
- Users can belong to one or more tenants with different roles.
- JWTs include `tenant_id` and the user's role within that tenant.
- A user authenticated for tenant A cannot access tenant B's subdomain (unless they also belong to tenant B).
- Superadmin users can access all tenants.
- The login endpoint on `acme.analytics.io` only accepts users who belong to the `acme` tenant.

### Verification Steps
1. Write pytest tests for multi-tenant user membership (user in one tenant, user in multiple tenants).
2. Test login flow: user logs in on `acme.analytics.io`, JWT contains `acme` tenant_id.
3. Test cross-tenant rejection: use `acme` JWT on `globex.analytics.io`, expect 403.
4. Test superadmin access across tenants.
5. Test user invitation flow within a tenant.

---

## Task 10: Update the Next.js Frontend for Multi-Tenancy

### Goal
Update the Next.js frontend to be tenant-aware, resolving tenant from the subdomain and passing tenant context with all API calls.

### Scope
- Implement tenant resolution on the frontend: extract subdomain from `window.location.hostname` and/or use Next.js middleware to resolve tenant on the server side.
- Store resolved tenant context in a React context provider (`TenantProvider`).
- Update the API client/fetch wrapper to automatically include the `Host` header (or `X-Tenant-ID` header) with all API requests.
- Update the login/signup pages to be tenant-branded (pull branding config from a `/api/tenant/config` endpoint).
- Update any frontend routing or navigation that hardcodes domain names.
- Handle the case where a user navigates to a non-existent tenant subdomain (show a "tenant not found" page).
- Update the Next.js config to handle wildcard subdomains (if using Vercel, configure `rewrites`; if self-hosted, configure reverse proxy rules).

### Dependencies
- Task 3 (Backend tenant resolution must be functional).
- Task 9 (Auth must be tenant-aware for login flows).

### Acceptance Criteria
- Visiting `acme.analytics.io` renders the app with acme's branding and data.
- Visiting `globex.analytics.io` renders the app with globex's branding and data.
- All API calls from the frontend include proper tenant identification.
- Visiting an unknown subdomain shows a "tenant not found" page.
- The tenant context is available throughout the React component tree.

### Verification Steps
1. Write Cypress tests that visit `acme.analytics.io` and verify tenant-specific branding elements.
2. Write Cypress tests that verify API calls include the correct tenant identification headers.
3. Test unknown subdomain handling in Cypress.
4. Verify Next.js middleware correctly resolves tenant on server-rendered pages.
5. Test that switching between tenants (for multi-tenant users) updates all context correctly.

---

## Task 11: Implement Tenant-Aware Database Indexing and Query Optimization

### Goal
Optimize database performance for multi-tenant queries by adding appropriate indexes and analyzing query plans.

### Scope
- Add composite indexes on `(tenant_id, <frequently_queried_column>)` for all high-traffic query patterns.
- Analyze the top 20 slowest queries (from existing query logs or `pg_stat_statements`) and optimize for multi-tenant access patterns.
- Consider and evaluate PostgreSQL Row-Level Security (RLS) as a defense-in-depth mechanism:
  - Create RLS policies on all tenant-scoped tables.
  - Set `current_setting('app.current_tenant_id')` at the start of each database connection.
  - RLS acts as a safety net; the ORM filter from Task 4 is the primary mechanism.
- Update `VACUUM` and `ANALYZE` schedules to account for increased table sizes.
- Evaluate partitioning by `tenant_id` for the largest tables (if any table is expected to exceed 100M rows). Document the decision.

### Dependencies
- Task 2 (`tenant_id` columns exist).
- Task 4 (ORM-level filtering is in place so we can benchmark).

### Acceptance Criteria
- Composite indexes exist for the top 20 query patterns.
- Query execution time for tenant-scoped queries is within 20% of single-tenant performance (benchmarked).
- RLS policies are active on all tenant-scoped tables (if the team decides to implement RLS).
- A partitioning decision is documented with supporting data.

### Verification Steps
1. Run `EXPLAIN ANALYZE` on the top 20 queries and confirm index usage.
2. Benchmark: run a standard query set against a test database loaded with data for 100 simulated tenants and compare latency to single-tenant baseline.
3. Test RLS: connect to the database with `app.current_tenant_id` set to tenant A and attempt to `SELECT` from tenant B's data -- expect zero rows.
4. Verify no sequential scans on tenant-scoped tables for common query patterns (check `pg_stat_user_tables`).

---

## Task 12: Implement Tenant-Scoped Rate Limiting and Resource Quotas

### Goal
Prevent any single tenant from monopolizing shared resources by implementing per-tenant rate limits and resource quotas.

### Scope
- Implement per-tenant API rate limiting using Redis (e.g., sliding window algorithm):
  - Default: 1000 requests/minute per tenant (configurable per plan tier).
  - Return `429 Too Many Requests` with `Retry-After` header when exceeded.
- Implement per-tenant resource quotas stored in the `tenants` table or a separate `tenant_quotas` table:
  - Max users per tenant.
  - Max storage per tenant (for file uploads, if applicable).
  - Max API calls per day.
  - Max concurrent Celery tasks.
- Create middleware that checks quotas before processing requests.
- Create an admin API to view and modify tenant quotas.
- Implement Celery task concurrency limits per tenant (e.g., using Redis-based semaphores).

### Dependencies
- Task 1 (Tenant model).
- Task 3 (Tenant resolution).
- Task 6 (Redis tenant namespacing).

### Acceptance Criteria
- Exceeding rate limits returns 429 with correct headers.
- Exceeding quotas returns 403 with a descriptive error message.
- Rate limits and quotas are configurable per tenant.
- Celery task concurrency is bounded per tenant.
- Admin API allows viewing/modifying quotas.

### Verification Steps
1. Write a load test (e.g., with `locust` or `pytest` + `httpx`) that exceeds rate limits and confirms 429 responses.
2. Test quota enforcement: create a tenant with max 5 users, attempt to create a 6th, expect rejection.
3. Test Celery concurrency limit: enqueue 10 tasks for a tenant with a limit of 3, confirm only 3 run concurrently.
4. Test admin quota modification API.

---

## Task 13: Update Test Infrastructure for Multi-Tenancy

### Goal
Update the pytest and Cypress test infrastructure to support multi-tenant testing patterns, including fixtures, factories, and isolation verification.

### Scope
- **pytest fixtures:**
  - Create a `tenant` fixture that creates a test tenant and sets up `TenantContext`.
  - Create a `second_tenant` fixture for cross-tenant isolation tests.
  - Create a `tenant_user` fixture that creates a user belonging to the test tenant.
  - Update the `db_session` fixture to use the tenant-scoped session.
  - Create a `superadmin` fixture for admin endpoint tests.
- **Test factories (e.g., using `factory_boy`):**
  - Update all model factories to accept and propagate `tenant_id`.
- **Cross-tenant isolation test suite:**
  - Create a dedicated test module `test_tenant_isolation.py` with parametrized tests that verify every model/endpoint pair does not leak data across tenants.
- **Cypress updates:**
  - Update Cypress config to support testing against multiple subdomains.
  - Create Cypress commands for tenant-scoped login.
  - Write e2e isolation tests.

### Dependencies
- Task 3 (Tenant context).
- Task 4 (Tenant-scoped queries).
- Task 5 (Endpoints updated).

### Acceptance Criteria
- All existing tests pass with the new fixtures.
- Cross-tenant isolation tests exist for every tenant-scoped model.
- Cypress can run e2e tests against different tenant subdomains.
- Test factories automatically set `tenant_id`.
- A CI job runs the full isolation test suite.

### Verification Steps
1. Run `pytest` and confirm all tests pass.
2. Run the isolation test suite and confirm all isolation checks pass.
3. Run Cypress tests against two test tenant subdomains and confirm both pass.
4. Review test coverage report and confirm tenant-scoped code paths are covered.

---

## Task 14: Implement Tenant Provisioning and Onboarding Automation

### Goal
Build the automated workflow for creating new tenants, including database setup, DNS/subdomain configuration, and initial data seeding.

### Scope
- Create a `provision_tenant` service function that:
  - Creates the tenant row in the `tenants` table.
  - Creates the initial admin user and sends an invitation email.
  - Seeds default data (e.g., default dashboard templates, report templates, default settings).
  - Configures the subdomain (if using a DNS API like Cloudflare, automate DNS record creation; otherwise, document the manual step).
  - Sets default quotas based on the selected plan.
- Create an admin API endpoint: `POST /admin/tenants/provision` that triggers the full provisioning workflow.
- Create a CLI command (e.g., `python manage.py provision_tenant --slug acme --admin-email admin@acme.com`) for manual provisioning.
- Implement a tenant deletion/archival workflow that soft-deletes the tenant, anonymizes PII, and retains data for a configurable grace period before hard deletion.

### Dependencies
- Task 1 (Tenant model).
- Task 9 (Auth system for creating the admin user).
- Task 12 (Quotas for setting defaults).

### Acceptance Criteria
- Provisioning a new tenant creates all necessary records and the tenant is immediately accessible.
- The admin user receives an invitation email.
- Default data is seeded correctly.
- Tenant deletion soft-deletes and the tenant is no longer accessible via subdomain.
- The CLI command works for manual provisioning.

### Verification Steps
1. Run the provisioning endpoint/CLI and verify the new tenant is accessible at its subdomain.
2. Verify the admin user can log in.
3. Verify default data (dashboards, settings) exists for the new tenant.
4. Test tenant deletion: soft-delete a tenant and confirm its subdomain returns 404.
5. Write pytest tests for the full provisioning and deletion workflows.

---

## Task 15: Implement Observability and Monitoring for Multi-Tenancy

### Goal
Add tenant-aware logging, metrics, and monitoring so that operators can diagnose issues and track resource usage per tenant.

### Scope
- Add `tenant_id` and `tenant_slug` to all structured log entries (use a logging filter or middleware that injects these fields).
- Create per-tenant metrics (using Prometheus, Datadog, or similar):
  - Request count per tenant.
  - Request latency (p50/p95/p99) per tenant.
  - Error rate per tenant.
  - Database query count/time per tenant.
  - Celery task count/duration per tenant.
  - Cache hit/miss rate per tenant.
- Create a tenant health dashboard (Grafana or similar) with alerts for:
  - Single tenant consuming >X% of total resources.
  - Tenant error rate spike.
  - Tenant approaching quota limits.
- Add tenant context to error reporting (Sentry or similar) so errors can be filtered by tenant.

### Dependencies
- Task 3 (Tenant context for injecting into logs/metrics).
- Task 7 (Celery tenant awareness for task metrics).

### Acceptance Criteria
- All log entries for tenant-scoped operations include `tenant_id`.
- Per-tenant metrics are emitted and queryable.
- A monitoring dashboard shows per-tenant resource usage.
- Alerts fire when a tenant exceeds resource thresholds.
- Errors in Sentry (or equivalent) are tagged with `tenant_id`.

### Verification Steps
1. Trigger API requests for two different tenants and verify log entries contain the correct `tenant_id`.
2. Query the metrics backend for per-tenant request counts and confirm accuracy.
3. Verify the monitoring dashboard loads and shows data for multiple tenants.
4. Trigger a controlled error and verify it appears in error reporting tagged with the correct tenant.

---

## Task 16: Infrastructure and Deployment Configuration

### Goal
Update deployment configuration and infrastructure to support the shared multi-tenant architecture, replacing the per-customer deployment model.

### Scope
- Update the reverse proxy (Nginx/Traefik/Cloudflare) to route all `*.analytics.io` subdomains to the single application instance.
- Update environment configuration: remove per-deployment environment variables and replace with a shared config that loads tenant-specific settings from the database.
- Update the CI/CD pipeline:
  - Remove per-customer deployment jobs.
  - Add a single deployment target.
  - Add the tenant isolation test suite as a required CI check.
- Update the database connection configuration to use connection pooling appropriate for multi-tenant load (e.g., PgBouncer with a pool size tuned for 500+ tenants).
- Update Redis configuration: ensure sufficient memory and configure maxmemory policies.
- Update Celery worker configuration: tune concurrency and prefetch settings for shared workload.
- Configure horizontal autoscaling rules for the FastAPI application and Celery workers based on request volume / queue depth.
- Document the new architecture with a diagram.

### Dependencies
- All previous tasks should be substantially complete before deploying to production.

### Acceptance Criteria
- All `*.analytics.io` subdomains route to the shared application.
- The application starts with a single shared configuration.
- CI/CD pipeline deploys a single instance and runs isolation tests.
- Connection pooling is configured and tested under load.
- Autoscaling rules are in place and tested.

### Verification Steps
1. Deploy to staging and verify 3+ test subdomains all route to the same instance.
2. Run a load test simulating 100 concurrent tenants and verify the system scales appropriately.
3. Verify CI pipeline runs and the isolation test suite is a required check.
4. Monitor connection pool utilization under load and confirm no exhaustion.
5. Verify autoscaling triggers when load exceeds thresholds.

---

## Task 17: Production Migration Runbook and Cutover Plan

### Goal
Create a detailed, step-by-step runbook for the production cutover from 23 single-tenant deployments to the shared multi-tenant deployment.

### Scope
- Document the migration sequence:
  1. Pre-migration: Full backup of all 23 databases. Verify backup integrity.
  2. Deploy the multi-tenant application to a new production environment.
  3. Run the data migration script (Task 8) to consolidate all data.
  4. Verify data integrity (row counts, FK integrity, spot checks).
  5. Update DNS: point all 23 subdomains to the new shared instance.
  6. Monitor for errors and performance issues (use Task 15 observability).
  7. Keep old deployments running in read-only mode for 48 hours as a rollback option.
  8. Decommission old deployments after the grace period.
- Define rollback procedures at each stage.
- Define success criteria and go/no-go checkpoints.
- Create a communication plan for notifying customers of the migration.
- Estimate downtime and plan for a maintenance window.
- Create a post-migration verification checklist.

### Dependencies
- All previous tasks (this is the final coordination task).

### Acceptance Criteria
- The runbook is detailed enough for an on-call engineer to execute without additional context.
- Rollback procedures are defined for every step.
- Success criteria are measurable and specific.
- Customer communication templates are prepared.
- The migration has been rehearsed at least once in a staging environment.

### Verification Steps
1. Peer review the runbook with at least two senior engineers.
2. Execute the full runbook in a staging environment and document any issues.
3. Time the staging migration to estimate production downtime.
4. Verify rollback procedures by executing a rollback in staging.
5. Review customer communication drafts with the product/support team.

---

## Dependency Graph

```
Task 1: Tenant Model (no dependencies)
  |
  +---> Task 2: Add tenant_id to tables (depends on: 1)
  |       |
  |       +---> Task 4: ORM-level filtering (depends on: 2, 3)
  |       |       |
  |       |       +---> Task 5: Update API endpoints (depends on: 3, 4)
  |       |       +---> Task 11: DB indexing & optimization (depends on: 2, 4)
  |       |
  |       +---> Task 8: Data migration script (depends on: 1, 2)
  |
  +---> Task 3: Tenant resolution middleware (depends on: 1)
  |       |
  |       +---> Task 4: ORM-level filtering (depends on: 2, 3)
  |       +---> Task 6: Redis cache scoping (depends on: 3)
  |       +---> Task 7: Celery tenant awareness (depends on: 3, 4)
  |       +---> Task 9: Auth multi-tenancy (depends on: 1, 3)
  |       |       |
  |       |       +---> Task 10: Frontend multi-tenancy (depends on: 3, 9)
  |       |       +---> Task 14: Tenant provisioning (depends on: 1, 9, 12)
  |       |
  |       +---> Task 15: Observability (depends on: 3, 7)
  |
  +---> Task 12: Rate limiting & quotas (depends on: 1, 3, 6)
  |
  +---> Task 13: Test infrastructure (depends on: 3, 4, 5)
  |
  +---> Task 16: Infrastructure & deployment (depends on: all substantially complete)
  |
  +---> Task 17: Production cutover runbook (depends on: all tasks)
```

## Suggested Execution Order

**Phase 1 -- Foundation (can be parallelized):**
- Task 1: Tenant Model
- Task 3: Tenant Resolution Middleware (can start after Task 1)

**Phase 2 -- Data Layer (sequential):**
- Task 2: Add `tenant_id` to All Tables
- Task 4: ORM-Level Tenant Filtering

**Phase 3 -- Application Layer (can be parallelized):**
- Task 5: Update API Endpoints
- Task 6: Redis Cache Scoping
- Task 7: Celery Tenant Awareness
- Task 9: Auth Multi-Tenancy

**Phase 4 -- Frontend and Optimization (can be parallelized):**
- Task 10: Frontend Multi-Tenancy
- Task 11: DB Indexing and Optimization
- Task 12: Rate Limiting and Quotas

**Phase 5 -- Quality and Operations (can be parallelized):**
- Task 13: Test Infrastructure
- Task 14: Tenant Provisioning
- Task 15: Observability

**Phase 6 -- Go Live (sequential):**
- Task 16: Infrastructure and Deployment
- Task 8: Data Migration (execute in staging, then production)
- Task 17: Production Cutover
