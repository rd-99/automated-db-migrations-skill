# Migration Strategies and Conflict Playbook

Use this reference when strategy selection, zero-downtime compatibility, drift, or concurrent branch work affects the task.

## Strategy Selection

### Versioned / change-based

Choose when exact SQL must be reviewed and replayed consistently across multiple environments. Goose is a good executor for ordered SQL/Go migrations and records applied versions. This is the default for shared environments in this repository.

Strengths: deterministic history, clear audit trail, offline review, controlled application. Costs: developers must design the transition correctly, and live drift invalidates the assumed starting state.

### Declarative / state-based

Choose when a canonical desired schema exists and the target can be inspected safely at plan time. Atlas calculates a transition from current to desired state.

Strengths: fast iteration and less handwritten DDL. Costs: the generated plan depends on actual state and may differ between environments; production needs reviewed/pre-approved plans and policy checks.

### Hybrid versioned authoring and independent verification

Use Atlas to calculate or inspect a proposed diff, commit reviewed versioned migration SQL, execute it with one owner, and use Atlas again to verify convergence. In the current repository, Goose should remain the sole executor/history owner while Atlas compares database states. If Atlas becomes the versioned executor, make that a deliberate migration of ownership rather than running both executors.

### Expand-and-contract

Use for rolling deployments, multiple application versions, large tables, or breaking changes.

1. **Expand:** add nullable columns, new tables, compatible indexes, or parallel APIs. Old and new code both work.
2. **Migrate:** backfill in bounded batches; dual-read/write only when necessary and with a defined source of truth.
3. **Switch:** deploy code that exclusively uses the new shape and verify adoption.
4. **Contract:** remove old columns, constraints, tables, or compatibility code in a later release after proving no old clients remain.

Examples:

- Rename a column by adding the new column, backfilling, temporarily supporting both, switching reads/writes, then dropping the old column later.
- Add `NOT NULL` by adding/using the column, backfilling nulls, validating a check constraint where supported, then enforcing non-null in a later phase.
- Replace a table by writing both or using a durable change-capture/backfill path, verifying parity, switching reads, and removing the old path later.

### Maintenance-window / offline

Acceptable for a bounded system when blocking/rewrite time is measured, downtime is approved, backups/restore are verified, and application traffic is stopped or fenced. It is not a shortcut around compatibility analysis.

### Baseline or squash

Use to adopt an existing schema or deliberately reduce bootstrap time. Record the baseline version and exact schema fingerprint. Continue testing upgrades from every still-supported deployed version; retain archived history for audit and recovery.

## Conflict Playbook

### Two branches choose the same Goose version

Before either reaches a shared database:

1. update the feature branch from the target branch;
2. keep the target branch migration unchanged;
3. assign the feature migration a new later version;
4. update dependent SQL/application changes;
5. replay all migrations on a clean database and from the target branch schema.

If either migration was applied to a shared database, its version/file is immutable. Add a new migration to reconcile the histories. Avoid Goose out-of-order mode unless the team has explicitly designed for it and has proven every supported upgrade path; chronological success is not proof of semantic safety.

### Separate versions change the same object incompatibly

Git may not report a conflict because the changes are in different files. Reconstruct the target branch schema, define one intended final state, and adjust only unapplied feature migrations or append a new reconciliation migration. Test object definitions, data preservation, and old/new application compatibility—not just whether SQL exits successfully.

### Atlas `atlas.sum` merge conflict or checksum error

- Do not choose one side of `atlas.sum` blindly.
- Rebase/merge the latest migration directory first.
- Use `atlas migrate rebase` if the feature migration must move after newly landed migrations.
- Resolve SQL and application-level conflicts.
- Run `atlas migrate hash` only after the directory is correct, then review and commit the checksum change.
- If an applied migration was edited, restore it and create a new corrective migration instead of hashing the edit.

### Manual live drift

Freeze deployment and capture an Atlas diff limited to managed schemas. Decide explicitly:

- **Repository is authoritative:** revert the manual change through an approved, auditable operation or migration.
- **Live change is intentional:** encode it as a new migration that also succeeds from a clean repository-built state. If live already has the effect, use guarded reconciliation only when it verifies the existing definition rather than silently accepting any object with the same name.

Then rerun fresh replay, live-version diff, live-clone forward apply, and final convergence diff. Never pipe Atlas-generated corrective SQL directly into production without review.

### Partially applied migration

1. Stop deploys and automated retry.
2. Determine whether the migration was transactional or `NO TRANSACTION`.
3. Inspect actual objects, locks, migration-history rows, and application traffic.
4. If the transaction rolled back, fix the unapplied migration only if it has never been recorded in any shared environment.
5. If effects remain, use a reviewed completion or compensating migration. Mark/force a version only after an authorized operator proves the database exactly matches that version's intended state.

### Concurrent migration runners

Use both CI concurrency and the executor/database lock. CI locking prevents duplicate jobs from the same automation system; a database lock also protects against other pipelines, operators, or application-startup migration runners. On lock loss or timeout, stop and inspect ownership rather than starting a competing writer.

## Rollback Decision

Choose among:

- **Application rollback:** safest when the expanded schema remains backward-compatible.
- **Forward fix:** default once a migration is applied and new data may depend on it.
- **Down migration:** only when tested against realistic data and known not to discard required writes.
- **Point-in-time restore:** disaster recovery, not routine deployment rollback; it can discard unrelated writes and requires an incident-level decision.

Document the choice before production apply for destructive, long-running, or non-transactional migrations.

## Official Tool References

- Goose annotations: <https://pressly.github.io/goose/documentation/annotations/>
- Goose commands: <https://pressly.github.io/goose/documentation/cli-commands/>
- Atlas declarative vs versioned workflows: <https://atlasgo.io/concepts/declarative-vs-versioned>
- Atlas schema diff: <https://atlasgo.io/declarative/diff>
- Atlas migration lint: <https://atlasgo.io/versioned/lint>
- Atlas migration directory integrity: <https://atlasgo.io/concepts/migration-directory-integrity>
