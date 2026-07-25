# Migration Safety

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · **Migration Safety** · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)

---

## Overview

Scans added lines in database migration files for **destructive or locking** DDL. Migrations are the one
category of change that cannot be rolled back by reverting the commit: once `DROP COLUMN` has run in
production, the data is gone.

Unlike the other checks in this category, its severity is **configurable**, so it can be a real gate
rather than advice.

### What it looks for

| Finding | Risk |
|---|---|
| `DROP TABLE` · `DROP COLUMN` · `TRUNCATE` | Irreversible data loss |
| `RENAME COLUMN` / `RENAME TABLE` | Breaks any code still reading the old name — needs a two-phase deploy |
| `DROP CONSTRAINT` / `DROP INDEX` | Silently removes an invariant or a performance guarantee |
| `ALTER COLUMN … SET NOT NULL` | Locks the table and fails outright if any existing row is `NULL` |
| `ADD COLUMN … NOT NULL` with no `DEFAULT` | The classic breakage: fine on an empty table, fails or locks on a populated one |
| `op.drop_column` · `op.drop_table` · `op.drop_constraint` | The same operations expressed through Alembic |

The `NOT NULL` without `DEFAULT` rule is a conjunction rather than a keyword match — the line must
contain `ADD COLUMN` **and** `NOT NULL` **and not** `DEFAULT`.

| Property | Value |
|---|---|
| Display name | `Migration Safety` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate migration-safety` |
| Config key | `migrationSafety` |
| Source | `src/core/checks/security/migration-safety.ts` |

## When it applies

Both conditions must hold:

1. `migrationSafety.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

The check passes immediately when the PR touches no migration file.

**It is deliberately not scoped by `sourcePath`.** Migrations commonly live outside `src` — in `db/`,
`alembic/`, or `prisma/` — so restricting them to the source path would silently disable the check for
most projects. `ignoreDirs` still applies.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `migrationSafety.enabled` | boolean | `true` | Set `false` to skip the check |
| `migrationSafety.severity` | `error` \| `warn` | `"warn"` | `error` **fails** the run; `warn` advises |
| `migrationSafety.paths` | string[] | see below | Globs treated as migrations. Setting this **replaces** the defaults |

Default `paths`:

```json
["**/migrations/**", "**/migrate/**", "**/db/migrate/**", "**/versions/**", "**/*.sql"]
```

`**/db/migrate/**` covers Rails and `**/versions/**` covers Alembic. Note that `**/*.sql` means *any*
SQL file counts as a migration, wherever it lives.

> `paths` is read by the check but is **not** declared in `pr-checkmate.schema.json`, so your editor
> will not autocomplete it.

### Examples

Make destructive DDL block the merge — a reasonable default for a production database:

```json
{
  "migrationSafety": { "severity": "error" }
}
```

Narrow the scope to a Prisma project, so ad-hoc analytics `.sql` files stop being scanned:

```json
{
  "migrationSafety": {
    "paths": ["prisma/migrations/**"],
    "severity": "error"
  }
}
```

## Disabling

```json
{ "migrationSafety": { "enabled": false } }
```

`migrationSafety.severity` and the universal `severity` map do the same job here; the universal one
wins, since it is applied to the outcome after the check returns.

## Working with a finding

The point of the check is to force the two-phase pattern for anything destructive. A rename, for
instance, is safe as *add new → backfill → switch reads → drop old across two deploys*, and unsafe as a
single `RENAME COLUMN`. When the migration genuinely is safe — a new table, an empty environment — accept
the specific line:

```sql
ALTER TABLE users DROP COLUMN legacy_flag; -- pr-checkmate-ignore — column added and unused in this release
```

## Notes

- Severity affects **both** the log level and the outcome: `error` logs with ❌ and returns `fail`,
  `warn` logs with ⚠️ and returns `warn`.
- The summary is compact — `2× DROP COLUMN, 1× SET NOT NULL` — built by truncating each label at its
  ` — ` explanation.
- The log shows at most **2 examples per finding label**; matched lines are trimmed to 120 characters.
- SQL comments are **not** stripped before matching, so a commented-out `DROP TABLE` can still be
  reported. Use the inline directive if that happens.
- Only added lines are scanned. Deleting a migration file is not reported here — that shows up in
  [Sensitive Files](sensitive-files.md) only if the path matches one of its categories.

---

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · **Migration Safety** · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)
