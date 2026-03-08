# Task Decomposition: MySQL-to-PostgreSQL Zero-Downtime Migration

## Context

- **Source**: RDS MySQL 8.0, single database containing a `users` table (12M rows)
- **Target**: RDS PostgreSQL 15 (no existing PostgreSQL infrastructure)
- **Consumers**: Main API (Node.js), Background Job Processor (Python/Celery), Admin Dashboard (Rails)
- **Foreign Key Dependencies**: `orders`, `sessions`, `audit_logs` all reference `users`
- **Constraint**: Zero downtime throughout migration

---

## Task 1: Provision and Configure RDS PostgreSQL Infrastructure

### Goal
Stand up the target RDS PostgreSQL 15 instance on AWS with networking, security groups, and parameter groups that mirror the availability and security posture of the existing MySQL setup.

### Scope
- Create an RDS PostgreSQL 15 instance (same VPC, availability zone strategy, and instance class as the MySQL instance).
- Configure security groups to allow inbound connections from all three consuming services.
- Set up a dedicated parameter group tuned for the workload (e.g., `work_mem`, `shared_buffers`, `max_connections` matching or exceeding current MySQL settings).
- Enable automated backups, encryption at rest (KMS), and enhanced monitoring.
- Create an application-level database user with least-privilege permissions.
- Validate connectivity from each service's subnet/security group.

### Dependencies
- None (first task).

### Acceptance Criteria
- RDS PostgreSQL 15 instance is running and reachable from all three service environments.
- Automated backups and encryption are enabled.
- A non-root application database user exists with `CREATE`, `SELECT`, `INSERT`, `UPDATE`, `DELETE` privileges on the target database.
- Parameter group values are documented.

### Verification Steps
1. `psql -h <pg-endpoint> -U <app_user> -d <db_name> -c "SELECT version();"` returns PostgreSQL 15.x.
2. Connectivity test succeeds from a host in each service's security group.
3. AWS Console confirms encryption, backups, and monitoring are active.

---

## Task 2: Translate the MySQL Schema to PostgreSQL

### Goal
Produce a PostgreSQL-compatible schema for the `users` table and all related tables (`orders`, `sessions`, `audit_logs`) that preserves data types, constraints, indexes, and foreign key relationships.

### Scope
- Extract the current MySQL schema (`SHOW CREATE TABLE` for `users`, `orders`, `sessions`, `audit_logs`).
- Map MySQL data types to PostgreSQL equivalents (e.g., `INT AUTO_INCREMENT` to `SERIAL`/`BIGSERIAL`, `DATETIME` to `TIMESTAMP`, `TINYINT(1)` to `BOOLEAN`, `ENUM` to PostgreSQL `ENUM` or `CHECK` constraint, `JSON` to `JSONB`).
- Translate all indexes (unique, composite, full-text) to PostgreSQL syntax.
- Translate all foreign key constraints, ensuring `ON DELETE`/`ON UPDATE` actions are preserved.
- Translate any triggers, stored procedures, or generated/virtual columns.
- Handle MySQL-specific defaults like `CURRENT_TIMESTAMP` and `ON UPDATE CURRENT_TIMESTAMP` (PostgreSQL requires a trigger for the latter).
- Produce a single, reviewable `.sql` migration file.

### Dependencies
- Task 1 (target instance must exist to validate the schema against).

### Acceptance Criteria
- A `schema.sql` file that can be applied cleanly to the PostgreSQL instance.
- All four tables exist with correct columns, types, constraints, indexes, and foreign keys.
- Any `ON UPDATE CURRENT_TIMESTAMP` behavior is replicated via a trigger.
- The schema is reviewed and approved by at least one engineer from each consuming service team.

### Verification Steps
1. Apply `schema.sql` to the PostgreSQL instance; it completes without errors.
2. `\dt` lists all four tables.
3. `\d users` shows columns, types, and constraints matching the original MySQL schema semantics.
4. Foreign key constraints are visible via `\d orders`, `\d sessions`, `\d audit_logs`.

---

## Task 3: Set Up Continuous Data Replication (MySQL to PostgreSQL)

### Goal
Establish continuous, near-real-time replication from MySQL to PostgreSQL so that the target database stays in sync with the source during the transition window.

### Scope
- Evaluate and select a replication tool (AWS DMS is the natural choice given the RDS-to-RDS scenario; alternatives include Debezium + Kafka or pgloader).
- Create an AWS DMS replication instance in the same VPC.
- Configure source (MySQL) and target (PostgreSQL) endpoints.
- Create a DMS migration task with:
  - **Full load** of all four tables (`users`, `orders`, `sessions`, `audit_logs`).
  - **CDC (Change Data Capture)** enabled for ongoing replication after the full load.
- Enable MySQL binary logging (`binlog_format=ROW`) if not already set (required for CDC).
- Handle sequence/auto-increment value synchronization: after full load, set PostgreSQL sequences to `MAX(id) + 1`.
- Configure error handling, logging, and CloudWatch alarms for replication lag and errors.

### Dependencies
- Task 1 (PostgreSQL instance exists).
- Task 2 (target schema is applied).

### Acceptance Criteria
- Full load completes for all four tables; row counts match between MySQL and PostgreSQL.
- CDC is active: a row inserted/updated/deleted in MySQL appears in PostgreSQL within 5 seconds under normal load.
- CloudWatch alarms are configured for replication lag > 10 seconds and task failure.
- PostgreSQL sequences are correctly set to avoid ID collisions.

### Verification Steps
1. Compare row counts: `SELECT COUNT(*) FROM users` on both databases.
2. Insert a test row in MySQL; confirm it appears in PostgreSQL within 5 seconds.
3. Update and delete a test row in MySQL; confirm changes propagate.
4. Check DMS task status in AWS Console: status is "Replication ongoing", no errors.
5. Verify sequences: `SELECT last_value FROM users_id_seq;` is >= `SELECT MAX(id) FROM users;`.

---

## Task 4: Audit and Adapt Application Queries for PostgreSQL Compatibility

### Goal
Identify every SQL query, ORM configuration, and database interaction across all three services and ensure they are compatible with PostgreSQL, producing a changeset for each service.

### Scope
- **Node.js Main API**:
  - Audit the ORM/query builder (likely Sequelize, Knex, or Prisma) for MySQL-specific syntax.
  - Replace backtick-quoted identifiers with double-quoted or unquoted identifiers.
  - Replace `IFNULL` with `COALESCE`, `LIMIT x, y` with `LIMIT y OFFSET x`, `NOW()` usage differences, `GROUP_CONCAT` with `STRING_AGG`, etc.
  - Update the database driver from `mysql2` to `pg`.
  - Update connection configuration (port 5432, SSL settings).
- **Python/Celery Background Jobs**:
  - Audit the ORM (likely SQLAlchemy or Django ORM) for MySQL-specific dialect usage.
  - Change the SQLAlchemy connection string from `mysql+pymysql://` to `postgresql+psycopg2://`.
  - Review raw SQL queries for MySQL-isms.
- **Rails Admin Dashboard**:
  - Switch the `database.yml` adapter from `mysql2` to `postgresql`.
  - Replace the `mysql2` gem with `pg` gem in the Gemfile.
  - Audit any raw SQL (`.find_by_sql`, `.execute`) for MySQL-specific syntax.
  - Review Active Record migrations for MySQL-specific column types.
- **Cross-cutting**:
  - Identify any use of MySQL-specific features: `INSERT ... ON DUPLICATE KEY UPDATE` (convert to `INSERT ... ON CONFLICT ... DO UPDATE`), `REPLACE INTO`, `FOUND_ROWS()`, etc.
  - Handle boolean representation differences (`TINYINT(1)` vs native `BOOLEAN`).
  - Handle case-sensitivity differences (MySQL is case-insensitive for string comparisons by default; PostgreSQL is case-sensitive). Add `CITEXT` extension or `LOWER()` calls where needed.

### Dependencies
- Task 2 (schema translation reveals the type mappings the application code must align with).

### Acceptance Criteria
- Each service has a branch/changeset with all query and configuration changes.
- All existing unit and integration tests pass when run against a local PostgreSQL instance.
- No MySQL-specific SQL syntax remains in any codebase (verified by grep).
- Connection configuration is externalized behind environment variables or feature flags so the database target can be switched without a code deploy.

### Verification Steps
1. Run each service's test suite against a local PostgreSQL 15 Docker container with the translated schema; all tests pass.
2. `grep -ri 'mysql\|backtick\|IFNULL\|GROUP_CONCAT\|ON DUPLICATE KEY' src/` returns no hits (adjusted per codebase).
3. Code review confirms connection strings are environment-driven.

---

## Task 5: Implement a Dual-Write / Feature-Flag Abstraction Layer

### Goal
Enable each service to write to both MySQL and PostgreSQL simultaneously during the transition period, controlled by a feature flag, so that the cutover can be performed gradually and rolled back instantly.

### Scope
- Introduce a database routing layer (or middleware) in each service that:
  - **Reads** from the **primary** database (initially MySQL).
  - **Writes** to **both** databases (MySQL first, then async to PostgreSQL) once dual-write mode is enabled.
  - Is controlled by a feature flag (e.g., LaunchDarkly, environment variable, or a simple config toggle).
- Implement a "shadow read" mode: optionally read from both databases and log discrepancies without affecting the response (for validation).
- Handle failure modes: if the PostgreSQL write fails, log the error but do not fail the request (MySQL remains the source of truth until cutover).
- Ensure write ordering and transaction semantics are preserved for critical operations.

### Dependencies
- Task 4 (queries must be PostgreSQL-compatible before dual-writing).
- Task 3 (DMS replication should be running so PostgreSQL is already populated before dual-writes start; dual-write supplements CDC to close the consistency gap).

### Acceptance Criteria
- Feature flag toggles dual-write on/off per service without a deploy.
- When dual-write is on, every write to MySQL is also applied to PostgreSQL.
- PostgreSQL write failures are logged and alerted on but do not cause user-facing errors.
- Shadow read mode is available and logs discrepancies to a structured log/metric.

### Verification Steps
1. Enable dual-write flag; perform CRUD operations via the API; confirm rows exist in both databases.
2. Simulate a PostgreSQL outage (revoke network access); confirm the API continues to function and errors are logged.
3. Enable shadow reads; perform read operations; confirm comparison logs are generated and report zero discrepancies.

---

## Task 6: Data Validation and Consistency Verification

### Goal
Confirm that the data in PostgreSQL is a complete and accurate replica of MySQL before cutting over reads.

### Scope
- Build (or configure) a row-level comparison tool that validates data consistency between MySQL and PostgreSQL.
  - Compare row counts per table.
  - Compare checksums or hash digests of sampled rows (e.g., every 1000th row by primary key).
  - Compare all rows for critical columns (e.g., `email`, `password_hash`, `status`).
- Run validation on all four tables: `users`, `orders`, `sessions`, `audit_logs`.
- Identify and reconcile any discrepancies (character encoding issues, timezone differences in timestamps, trailing whitespace, NULL vs empty string).
- Validate that auto-increment/sequence values in PostgreSQL are ahead of the max ID to prevent collisions.
- Document any intentional data transformations (e.g., `TINYINT(1)` stored as `0/1` in MySQL mapped to `true/false` in PostgreSQL).

### Dependencies
- Task 3 (replication must be running and caught up).
- Task 5 (dual-write should be active to keep both databases in sync during validation).

### Acceptance Criteria
- Row counts match exactly across all four tables.
- A sampled row-level comparison (minimum 10% of rows) shows zero discrepancies.
- Sequence values in PostgreSQL exceed the current max IDs.
- A validation report is produced and reviewed.

### Verification Steps
1. Run the comparison tool; output shows 0 discrepancies for all tables.
2. `SELECT COUNT(*) FROM users` matches on both databases.
3. `SELECT MAX(id) FROM users` on MySQL < `SELECT last_value FROM users_id_seq` on PostgreSQL.
4. Spot-check 10 random user records manually: all fields match.

---

## Task 7: Cut Over Reads to PostgreSQL

### Goal
Switch all read traffic from MySQL to PostgreSQL, while keeping MySQL as the write target (fallback safety net).

### Scope
- Update the feature flag / database routing layer to direct reads to PostgreSQL.
- Roll out incrementally:
  1. 5% of read traffic to PostgreSQL (canary).
  2. Monitor error rates, latency (p50, p95, p99), and query plans for 1 hour.
  3. 25% of reads to PostgreSQL; monitor for 2 hours.
  4. 100% of reads to PostgreSQL; monitor for 24 hours.
- Set up dashboards and alerts comparing PostgreSQL read latency to the MySQL baseline.
- Have a rollback plan: flip the flag back to MySQL reads instantly if error rates exceed thresholds.

### Dependencies
- Task 6 (data must be validated and consistent).
- Task 5 (routing layer must support read-source switching).

### Acceptance Criteria
- 100% of reads are served from PostgreSQL.
- p99 read latency is within 10% of the MySQL baseline.
- Error rate is at or below the pre-migration baseline.
- No user-reported issues for 24 hours.

### Verification Steps
1. Application logs/metrics confirm all read queries hit PostgreSQL.
2. Latency dashboards show p99 within acceptable range.
3. Error rate dashboards show no increase.
4. Canary/synthetic monitoring tests pass.

---

## Task 8: Cut Over Writes to PostgreSQL and Decommission Dual-Write

### Goal
Make PostgreSQL the sole primary database for both reads and writes. Reverse the replication direction so MySQL receives changes from PostgreSQL (for rollback safety), then eventually decommission MySQL.

### Scope
- **Phase 1 -- Write cutover**:
  - Stop DMS CDC replication (MySQL -> PostgreSQL direction).
  - Switch the feature flag so writes go to PostgreSQL as primary.
  - Keep dual-write active in reverse (PostgreSQL primary, MySQL secondary) for rollback safety.
  - Monitor write latency, error rates, and data integrity for 24-48 hours.
- **Phase 2 -- Reverse replication (optional safety net)**:
  - Configure DMS or logical replication from PostgreSQL to MySQL so the old database stays current for emergency rollback.
  - Monitor replication lag.
- **Phase 3 -- Decommission MySQL**:
  - After a bake period (e.g., 7-14 days) with zero incidents:
    - Disable reverse dual-write / reverse replication.
    - Take a final snapshot of the MySQL RDS instance.
    - Notify all teams that MySQL is being decommissioned.
    - Stop and delete the DMS replication instance.
    - Set the MySQL RDS instance to "stopped" (keep for 30 days as insurance), then delete.
  - Remove MySQL drivers, connection strings, and dual-write code from all services.
  - Remove the feature flag.

### Dependencies
- Task 7 (reads must be fully on PostgreSQL and stable).

### Acceptance Criteria
- PostgreSQL is the sole primary database for all reads and writes.
- Reverse replication or reverse dual-write to MySQL is active during the bake period.
- After the bake period, MySQL is decommissioned (snapshot retained).
- All dual-write and feature-flag code is removed from all three services.
- DMS replication instance is deleted.

### Verification Steps
1. Application logs confirm all reads and writes go to PostgreSQL.
2. MySQL receives reverse-replicated changes during the bake period.
3. After decommission: `aws rds describe-db-instances` no longer lists the MySQL instance (or shows it as stopped/deleted).
4. Codebase grep for MySQL references returns no hits.
5. Cost monitoring confirms MySQL RDS charges have stopped.

---

## Task 9: Performance Benchmarking and Query Optimization on PostgreSQL

### Goal
Ensure PostgreSQL performs at or above the MySQL baseline for all critical queries, and optimize any regressions.

### Scope
- Identify the top 20 most-frequent and top 20 slowest queries from MySQL slow query log / Performance Insights.
- Run each query against PostgreSQL using `EXPLAIN ANALYZE` and compare execution plans and timing to MySQL.
- Add or adjust indexes on PostgreSQL where query plans show sequential scans on large tables.
- Tune PostgreSQL configuration parameters (`random_page_cost`, `effective_cache_size`, `work_mem`) based on observed query patterns.
- Enable `pg_stat_statements` extension for ongoing query performance monitoring.
- Load-test the PostgreSQL instance with realistic traffic (using a tool like pgbench, k6, or Locust) to validate throughput under peak load.

### Dependencies
- Task 2 (schema must exist).
- Task 3 (data must be loaded).
- Should be completed before Task 7 (read cutover).

### Acceptance Criteria
- All top-20 frequent queries have execution times within 10% of MySQL baseline.
- All top-20 slow queries are equal to or faster than MySQL.
- `pg_stat_statements` is enabled and collecting data.
- Load test results show the PostgreSQL instance handles peak traffic with p99 latency under the defined SLA.

### Verification Steps
1. Side-by-side comparison spreadsheet of query timings (MySQL vs PostgreSQL) for top-40 queries.
2. `EXPLAIN ANALYZE` output for critical queries shows index scans, not sequential scans on large tables.
3. Load test report shows throughput and latency within acceptable bounds.
4. `SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;` returns data.

---

## Task 10: Update Monitoring, Alerting, and Runbooks

### Goal
Ensure all operational tooling (monitoring, alerting, runbooks, dashboards) is updated to reflect PostgreSQL as the primary database.

### Scope
- Create or update CloudWatch dashboards for RDS PostgreSQL metrics (CPU, memory, IOPS, connections, replication lag during bake period).
- Update PagerDuty / OpsGenie / alerting rules to point to PostgreSQL metrics.
- Update application-level database health checks in each service to check PostgreSQL connectivity.
- Update runbooks for:
  - Database failover procedures (RDS Multi-AZ).
  - Backup and restore procedures.
  - Connection pool troubleshooting.
  - Slow query investigation (using `pg_stat_statements` instead of MySQL slow query log).
- Update any CI/CD pipelines that run integration tests against a database to use PostgreSQL.
- Update infrastructure-as-code (Terraform, CloudFormation, CDK) to reflect the new PostgreSQL instance and remove MySQL resources.

### Dependencies
- Task 1 (PostgreSQL instance must exist for dashboards).
- Should be completed before Task 7 (read cutover) so that monitoring is in place before production traffic hits PostgreSQL.

### Acceptance Criteria
- CloudWatch dashboards exist for all critical PostgreSQL metrics.
- Alerts fire correctly (tested with a simulated threshold breach).
- Health check endpoints in all three services verify PostgreSQL connectivity.
- Runbooks are updated and reviewed by the on-call team.
- CI/CD pipelines use PostgreSQL for integration tests.
- IaC reflects the current state of infrastructure.

### Verification Steps
1. CloudWatch dashboard loads and shows live PostgreSQL metrics.
2. Trigger a test alert (e.g., temporarily lower a CPU threshold); confirm it fires and routes correctly.
3. Stop PostgreSQL temporarily in a staging environment; confirm health checks fail and alerts fire.
4. CI/CD pipeline runs integration tests against PostgreSQL successfully.
5. `terraform plan` (or equivalent) shows no drift.

---

## Execution Order Summary

The tasks have the following dependency graph and recommended execution order:

```
Task 1: Provision PostgreSQL
  |
  v
Task 2: Translate Schema --------+
  |                               |
  v                               v
Task 3: Set Up Replication    Task 4: Adapt Application Queries
  |                               |
  |   +---------------------------+
  |   |
  v   v
Task 5: Dual-Write Layer     Task 9: Performance Benchmarking
  |                               |
  v                               |
Task 6: Data Validation          |
  |                               |
  v                               v
Task 7: Cut Over Reads  <--------+
  |
  v                         Task 10: Update Monitoring (do before Task 7)
Task 8: Cut Over Writes & Decommission MySQL
```

**Parallelizable work**:
- Tasks 3 and 4 can be done in parallel (after Task 2).
- Task 9 can run in parallel with Tasks 5 and 6 (after Task 3).
- Task 10 can be started as soon as Task 1 is complete and should be finished before Task 7.

**Estimated total timeline**: 4-6 weeks, depending on team capacity and the complexity of query adaptations discovered in Task 4.

---

## Risk Register

| Risk | Mitigation |
|------|------------|
| Query performance regression on PostgreSQL | Task 9 benchmarks queries before cutover; indexes and config are tuned. |
| Data inconsistency between MySQL and PostgreSQL | Task 6 validates data; dual-write + CDC provides belt-and-suspenders sync. |
| Application errors from untranslated MySQL-specific SQL | Task 4 audits all queries; test suites run against PostgreSQL before cutover. |
| Downtime during cutover | Feature-flag-driven routing (Task 5) enables instant rollback; no DNS or restart required. |
| Replication lag causes stale reads | CloudWatch alarms on DMS lag (Task 3); read cutover is gated on lag < threshold. |
| Rollback needed after write cutover | Reverse replication keeps MySQL current during bake period (Task 8). |
| Case-sensitivity differences cause bugs | Explicitly addressed in Task 4; `CITEXT` or `LOWER()` applied where needed. |
