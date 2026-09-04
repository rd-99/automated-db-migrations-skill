# Provider-Neutral CI/CD Blueprint

Use this reference when creating or reviewing migration automation. Translate platform syntax, but preserve the gates and state model.

## Generalized Configuration Keywords

Use neutral names in designs and templates, then map them to the selected platform:

| Keyword | Meaning |
| --- | --- |
| `CI_PROVIDER` | GitHub Actions, GitLab CI, Jenkins, Buildkite, or another runner |
| `SOURCE_REF` / `BASE_REF` | proposed source revision and target branch/revision |
| `MIGRATION_CHANGE_GLOB` | path filter that wakes migration validation/deployment |
| `RELEASE_SIGNAL` | version, manifest tag, release record, or immutable commit digest |
| `MIGRATIONS_DIR` | ordered migration files |
| `DB_DIALECT` / `DB_VERSION` | target engine and compatible temporary database version |
| `TARGET_DATABASE_URL` | secret connection value for the selected environment |
| `TEMP_DATABASE_URL` | disposable isolated validation database |
| `MANAGED_SCHEMAS` | schemas whose ownership and drift are enforced |
| `MIGRATION_EXECUTOR` | Goose or one deliberately selected history owner |
| `SCHEMA_ANALYZER` | Atlas or another independent diff/lint tool |
| `ARTIFACT_STORE` | container registry, object store, or package registry |
| `DEPLOY_TARGET` | cluster, VM, platform service, or on-prem host |
| `DEPLOY_CONCURRENCY_KEY` | one serialization key per target database/environment |
| `HEALTHCHECK_URL` | post-deploy service/schema compatibility probe |

Provider examples such as OIDC, AWS credentials, S3, ECR, SSM, IIS, or a particular runner label are adapters for these roles, not requirements of the migration design.

## Pull/Merge Request Validation

Run on migration-directory changes and on files that define the desired schema. Also allow manual dispatch for recovery and diagnosis, but do not make production mutation a side effect of a pull-request workflow.

### 1. Static and history checks

- reject duplicate versions, malformed filenames, and invalid Goose annotations;
- reject deletion or modification of versions already present on the target branch or applied to a shared database;
- verify any integrity file such as `atlas.sum`;
- lint risky operations and flag explicit exceptions for review;
- check that tool and database versions are pinned.

Do not rely only on `git diff --diff-filter=A` to decide whether migrations should run. It can miss pending files after squash merges, reruns, skipped deployments, force-updated refs, or changes spanning multiple commits. Let the migration history table determine what is pending; use path diffs only to avoid unnecessary CI setup.

### 2. Fresh replay

Start a disposable database matching `DB_DIALECT` and `DB_VERSION`, install required extensions/prerequisites, and run:

```sh
goose -dir "$MIGRATIONS_DIR" "$DB_DIALECT" "$TEMP_DATABASE_DSN" status
goose -dir "$MIGRATIONS_DIR" "$DB_DIALECT" "$TEMP_DATABASE_DSN" up
goose -dir "$MIGRATIONS_DIR" "$DB_DIALECT" "$TEMP_DATABASE_DSN" status
```

This proves new installations converge and catches syntax, ordering, dependency, and missing-object failures.

### 3. Reconstruct the repository state at the live version

Read the live migration version with a read-only credential. Fail closed if the history table is absent or ambiguous unless a reviewed baseline/adoption procedure is in progress. Apply repository migrations only through that version to a second disposable database.

Compare:

```text
actual live schema -> repository schema reconstructed at live recorded version
```

Use Atlas `schema diff` with explicit `--from`, `--to`, `--dev-url`, and managed-schema filters. Normalize only documented, stable provider noise. Never discard a diff merely because it is inconvenient.

### 4. Prove the forward path from reality

Copy schema-only definitions for `MANAGED_SCHEMAS` and the migration history rows from live into a third isolated database. Install any required extensions in a deterministic bootstrap step. Apply pending Goose migrations to this clone, then compare:

```text
live-schema clone after pending migrations -> clean database after all repository migrations
```

Copy no application rows unless they are needed to validate a data migration. When real data is required, use a sanitized representative fixture or an approved isolated snapshot; never expose production data to an untrusted runner.

### 5. Classify the result

| Live vs repo at live version | Live vs repo full | Forward clone converges | Classification | Action |
| --- | --- | --- | --- | --- |
| equal | equal | n/a | `IN_SYNC` | pass |
| equal | different | yes | `PENDING_REPO_MIGRATIONS` | pass validation; deploy applies pending files |
| different | equal | n/a | `LIVE_MATCHES_AFTER_PENDING` | pass only after verifying history/reconciliation intent |
| different | different | yes | `PARTIAL_PENDING_STATE` | pass only when pending history safely explains and converges the state |
| any | any | no | `MANUAL_DRIFT_OR_CONFLICT` | block and reconcile |

If an Atlas command itself fails, classify it as `CHECK_FAILED`, not as an empty diff.

## Deployment Workflow

1. Resolve `RELEASE_SIGNAL` and build an immutable artifact.
2. Validate required environment configuration and secret availability without printing credentials.
3. Acquire `DEPLOY_CONCURRENCY_KEY`; allow only one migration/deployment writer per target database.
4. Read Goose status and apply all pending migrations, rather than only files detected in the triggering commit.
5. On success, record the resulting version and promote/deploy the artifact in the strategy-specific order.
6. Run application and schema compatibility health checks.
7. Preserve logs, Atlas reports, migration versions, artifact digest, and target environment for audit.

For expand-and-contract, represent expansion, application switch/backfill, and contraction as distinct release gates. A single green workflow must not collapse the compatibility window.

## Credentials and Trust Boundaries

- Prefer short-lived workload identity over long-lived cloud keys.
- Separate read-only drift-check credentials from migration-writer credentials.
- Keep production secrets out of workflows triggered by untrusted fork code.
- Scope network access, database grants, managed schemas, environments, and approval rules narrowly.
- Pass connection strings as opaque secrets to clients. Avoid hand-parsing DSNs or rebuilding URLs in shell, which can break on escaped characters and leak values.
- Mask temporary signed URLs and avoid logging SQL containing sensitive values.

## Failure and Retry Boundaries

- Always clean up disposable databases, including after failure.
- Keep logs sufficient to distinguish tool failure, schema conflict, lock contention, and partial non-transactional execution.
- Retry only proven transient setup/read operations with a bounded count. Do not automatically rerun a failed migration until its transactional state and side effects are known.
- Never continue artifact promotion or application rollout after a required migration gate fails.
