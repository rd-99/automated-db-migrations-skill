---
name: automated-db-migrations
description: Design, implement, or review automated relational database migrations in CI/CD, including Goose execution, Atlas schema drift checks, deployment ordering, rollout strategies, and migration conflict recovery. Use for migration SQL or migration pipelines; do not use for ordinary application queries or seed-only data changes.
license: Proprietary
compatibility: agent
metadata:
  repo: gin-backend-core
  domain: database-delivery
---

# Automated Database Migrations

Build a repeatable path from reviewed schema change to verified environment state. Keep provider-specific delivery mechanics replaceable; the durable design is the migration history, validation gates, rollout order, and conflict policy.

## Start With the Existing System

Before editing, identify:

- database dialect/version and every schema managed by the repository;
- migration directory, naming convention, history table, and current live version;
- CI provider, protected branches/environments, release signal, deployment concurrency, and database network path;
- which tool owns migration execution and history;
- application compatibility requirements during rolling or mixed-version deploys.

Inspect repository-local instructions and existing migrations before choosing syntax. When working in this workspace, read [references/repository-patterns.md](references/repository-patterns.md) for the concrete examples and their generalizable equivalents.

## Choose a Strategy Explicitly

Use [references/strategies-and-conflicts.md](references/strategies-and-conflicts.md) when selecting a rollout model or resolving any conflict or drift.

- Prefer **versioned, forward-only migrations** for shared environments: reviewed files are immutable after application, Goose records versions, and corrective changes are new migrations.
- Use **declarative/state-based migrations** when a canonical desired schema exists and the organization accepts plan-time inspection and approval. Atlas can plan the transition, but production plans still require review and policy gates.
- Prefer a **hybrid** in this repository: Goose owns ordered execution and its version table; Atlas independently compares reconstructed, live, and intended schema states. Do not let two tools independently apply the same migration directory or maintain competing histories.
- Use **expand-and-contract** for rolling, zero-downtime, or backward-incompatible changes. Add compatible structures first, deploy tolerant application code, backfill/switch traffic, and remove old structures in a later release.
- Use a **maintenance-window/offline** migration only when the accepted downtime and rollback plan are explicit.
- Use a **baseline/snapshot** only to adopt a legacy database or deliberately compact history. Preserve an auditable boundary and test both new installs and upgrades from supported versions.

## Author Migrations Safely

- Allocate a unique monotonic version. Never renumber, rename, or edit a migration known to be applied in any shared environment.
- For Goose SQL files, put `-- +goose Up` on its own unindented line. Put the optional `-- +goose Down` section after it. Use `StatementBegin`/`StatementEnd` for functions or blocks containing internal semicolons.
- Keep the default transaction unless the database operation forbids it, such as PostgreSQL `CREATE INDEX CONCURRENTLY`. Mark a no-transaction migration explicitly and design it for safe inspection/resumption because partial effects can remain.
- Make the forward path safe and deterministic. `IF EXISTS`/`IF NOT EXISTS` may help convergence, but do not use them to hide an unexpected definition, ownership, type, or constraint mismatch.
- Separate large backfills from blocking DDL. Batch, checkpoint, observe, and make retries idempotent. Do not keep a long data rewrite inside a schema transaction by default.
- Treat `Down` as documentation and a tested option, not a promise. If rollback would destroy post-migration data, prefer application rollback plus a new forward corrective migration.
- Check lock duration, table rewrites, index build mode, replication impact, available disk, statement timeout, and compatibility with both old and new application versions.

## Build the CI Gates

Use [references/ci-blueprint.md](references/ci-blueprint.md) to implement or review a provider-neutral pipeline.

At minimum, CI must:

1. validate naming, ordering, Goose parsing, and duplicate versions;
2. replay all migrations into a clean temporary database;
3. reconstruct repository state through the live recorded version;
4. compare that state with the live schema using Atlas;
5. clone only the managed live schemas plus migration history into an isolated database, apply pending migrations with Goose, and compare the result with the fully migrated clean database;
6. classify expected pending changes separately from manual drift;
7. block on parse/apply failures, unresolved drift, destructive policy violations, or ambiguous state.

Add Atlas migration linting when available and licensed, but keep a replay-and-diff gate as the portable baseline. Pin Goose, Atlas, database images, and reusable CI actions to reviewed versions; update them deliberately.

## Order Deployment Around Compatibility

- Serialize migration application per target database with CI concurrency controls and a database-level migration lock where supported.
- Build artifacts early if useful, but gate artifact promotion and application rollout on the required schema phase.
- For additive/compatible changes, migrate then deploy the application.
- For expand-and-contract, deploy the phases across separate releases; never run the contract step until old application versions and old readers/writers are gone.
- Run `goose status` before and after apply, emit the applied version without secrets, and perform a schema-aware application health check.
- Do not blindly retry a failed migration. First determine whether it was transactional, whether partial effects remain, and whether another runner owns the lock.
- Require explicit authorization immediately before changing a shared or production database. A CI design or dry run does not authorize applying it.

## Handle Conflicts Without Rewriting History

- **Duplicate version before merge:** update from the target branch, assign the later migration a new version, and replay the combined history. Do not enable out-of-order execution as the default escape hatch.
- **Semantic conflict before merge:** determine the intended final schema, rewrite only migrations that have never reached a shared environment, then test both fresh replay and upgrade from the target branch state.
- **Applied migration changed/checksum mismatch:** restore the applied file exactly and add a new corrective migration. Do not recalculate an integrity hash merely to bless rewritten history.
- **Manual live drift:** block deployment. Decide whether repository history or the live change is authoritative; then either revert the live change through an approved operation or add a reconciliation migration and prove clean convergence on both fresh and live-clone paths.
- **Partial failure:** stop automated retries, capture the exact database state and migration logs, and have an authorized operator choose rollback, completion, or a corrective migration. Never mark a version applied merely to unblock CI.
- **Atlas integrity-file conflict:** rebase onto current migration history, resolve schema semantics, use `atlas migrate rebase` when appropriate, then regenerate `atlas.sum` with `atlas migrate hash`. Review the resulting SQL and hash change.

## Delivery Output

When implementing or reviewing automation, report:

- selected migration/rollout strategy and why;
- executor and schema-analysis roles (for example Goose executes, Atlas diffs/lints);
- environments and managed schemas in scope;
- validation states exercised: fresh, live-version, live-clone-plus-pending, and fully migrated;
- deployment ordering, concurrency/locking, rollback or roll-forward plan;
- conflicts or drift found and the exact stopping condition;
- commands/tests run and any checks that require trusted CI or database access.
