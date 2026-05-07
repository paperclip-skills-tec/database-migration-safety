---
name: database-migration-safety
description: Safe database migration workflow — naming conventions, idempotency, rollback verification, cross-branch cherry-picks, and CI validation. Use this skill whenever you are creating a new migration file, resolving migration conflicts between branches, cherry-picking a migration from another branch, verifying CI passes for migrations, debugging migration ordering issues, or reviewing a PR that touches the database schema. If you are unsure whether a change involves a migration, check — this skill's pre-flight checks prevent the most common QA blockers.
---

# Database Migration Safety

This skill guides you through safe database migration practices for the Paperclip AI project. Migrations are high-risk changes: a missing file, a duplicate prefix, or a missing rollback has cascading effects on every branch and CI environment. Follow these steps carefully.

## Pre-flight Checks (run before creating or applying any migration)

1. **List existing migration files and identify the next safe prefix.**
   ```bash
   ls migrations/ | sort
   ```
   The prefix must be a timestamp or sequential integer that is strictly greater than every existing prefix. For timestamp-style prefixes (`YYYYMMDDHHMMSS_`), use `date +%Y%m%d%H%M%S`. For integer prefixes (`NNN_`), find the highest existing number and add 1.

2. **Check for conflicts on other active branches.**
   ```bash
   git fetch origin
   git log --oneline --all -- migrations/ | head -20
   ```
   If another branch has an uncommitted or recently merged migration at the same prefix, stop — coordinate with that branch's author to agree on ordering before you create yours. Evidence: [TEC-385](/TEC/issues/TEC-385) (duplicate `003_` prefix causing conflicts).

3. **Confirm the migration tooling is available.**
   ```bash
   npm run migrate:status   # or equivalent — confirms the tool can connect
   ```
   If the command fails, diagnose the environment before proceeding.

## Creating a New Migration

### Required structure

Every migration file must have both an `up` and a `down` function. A migration without a `down` function is not mergeable.

```js
// Example Node.js / knex style
exports.up = async (knex) => {
  // ...
};

exports.down = async (knex) => {
  // Reverse the up function exactly
};
```

### Idempotency

Write `up` and `down` so they can be re-run safely:

- **Creating a table:** `CREATE TABLE IF NOT EXISTS`
- **Dropping a table:** `DROP TABLE IF EXISTS`
- **Adding a column:** check `information_schema.columns` before `ALTER TABLE ADD COLUMN`
- **Creating an index:** `CREATE INDEX IF NOT EXISTS`
- **Dropping an index:** `DROP INDEX IF EXISTS`

Why: migrations may be replayed during CI, staging reset, or local dev teardown. Non-idempotent migrations cause failures that are hard to diagnose.

### Index naming conventions

Name indexes explicitly — never rely on auto-generated names, which differ across database engines:

```sql
CREATE INDEX idx_<table>_<columns> ON <table>(<col1>, <col2>);
```

Example: `idx_users_email` for an index on `users.email`.

### Foreign keys

Every new foreign key column needs an index. The query planner and CASCADE operations require it for acceptable performance at scale:

```sql
ALTER TABLE orders ADD COLUMN user_id INTEGER REFERENCES users(id);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Large-table ALTER caution

For tables with more than ~100k rows, a naive `ALTER TABLE ADD COLUMN` acquires a full table lock. Strategies:

- Add the column as nullable first, then backfill, then add the constraint.
- Use `ADD COLUMN ... DEFAULT ...` only if the DB engine handles it online (PostgreSQL 11+).
- Document the expected row count and chosen strategy in the migration comment.

## SQL Safety (required — not optional)

**Never use string interpolation in SQL.** Use parameterized queries for all user-controlled or externally sourced values:

```js
// WRONG — SQL injection risk
knex.raw(`SELECT * FROM users WHERE id = ${userId}`);

// RIGHT
knex.raw('SELECT * FROM users WHERE id = ?', [userId]);
```

This is a BLOCKING item on the code review checklist (see [sdlc-process](/TEC/issues/TEC-792)).

## Testing the Rollback Locally

Before pushing, verify the `down` function works:

```bash
npm run migrate:latest   # apply up
npm run migrate:rollback # apply down
npm run migrate:latest   # apply up again — confirms idempotency
```

If any of these fail, fix the migration before pushing.

## CI Validation

Run the CI migrate check before committing:

```bash
npm run migrate:ci
```

This runs migrations against a fresh test database (no existing state). It catches:
- Missing files that exist locally but weren't committed
- Ordering errors that only appear in a clean environment
- Syntax errors that pass local lint but fail on the CI database engine version

**Do not push a migration without passing `migrate:ci` locally.** Evidence: [TEC-1115](/TEC/issues/TEC-1115) — a missing migration file that existed locally but was absent on the feature branch blocked QA on [TEC-1081](/TEC/issues/TEC-1081) and required a manual cherry-pick.

## Cross-Branch Migration Handling

### When cherry-picking a migration from another branch

1. Identify the exact commit SHA that added the migration file:
   ```bash
   git log --all --oneline -- migrations/<filename>
   ```

2. Cherry-pick onto your branch:
   ```bash
   git cherry-pick <sha>
   ```

3. Verify ordering — after cherry-pick, re-list the migrations directory and confirm the file sorts correctly relative to your branch's other migrations:
   ```bash
   ls migrations/ | sort
   ```

4. Re-run `npm run migrate:ci` to confirm the combined migration set is still valid.

5. Comment on the source issue to record that the cherry-pick was applied. Example: "Cherry-picked migration `<filename>` (SHA `<sha>`) onto `feature/TEC-NNN` — verified ordering and CI pass."

### When resolving a migration ordering conflict between branches

If two branches add migrations with conflicting prefixes:

1. Decide which migration logically comes first (usually the one that the other depends on, or the one that was merged to `main` first).
2. Rename the file with the later logical order to use a higher prefix, updating the filename and any internal reference.
3. Update the commit message to reference the conflict resolution.
4. Both sides re-run `migrate:ci` after renaming.

### Lockfile / dependency ordering

If migration A creates a table that migration B alters, B must have a higher prefix than A. Verify this in code review whenever you see an `ALTER TABLE` or `ADD COLUMN` that references a table added in a recent migration.

## Pre-PR Checklist

Before setting a migration-touching PR to `in_review`:

- [ ] Migration file has a unique, ordered prefix
- [ ] Both `up` and `down` are implemented
- [ ] `up` and `down` are idempotent (IF NOT EXISTS / IF EXISTS guards)
- [ ] New indexes on all new foreign key columns
- [ ] No string interpolation in SQL
- [ ] Large-table ALTER strategy documented if applicable
- [ ] `npm run migrate:rollback` passes locally
- [ ] `npm run migrate:ci` passes locally
- [ ] Migration file is committed on the feature branch (not just staged locally)

## Common Pitfalls

| Pitfall | Why it happens | Prevention |
|---|---|---|
| Missing migration file on branch | File created in a different worktree or never staged | Always `git status` in the exact worktree you ran `migrate:ci` from; commit before switching |
| Duplicate prefix | Two agents create migrations concurrently | Pre-flight check: list migrations on all active branches before choosing a prefix |
| Broken rollback | `down` written quickly, never tested | Run `migrate:rollback` locally before every push |
| SQL injection via string concat | Copy-paste from raw SQL exploratory work | All DB-facing values through parameterized queries; no exceptions |
| Missing index on FK | FK added in a hurry | Index immediately below every FK `ALTER TABLE` statement |
| Non-idempotent migration fails CI reset | `IF NOT EXISTS` omitted | Template: start every DDL statement with the idempotency guard |
| Ordering conflict after merge | Two branches with sequential numeric prefixes | Check `git log --all -- migrations/` before creating a new file |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*