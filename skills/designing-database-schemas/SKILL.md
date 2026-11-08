---
name: designing-database-schemas
description: Use when designing or changing a relational database schema (PostgreSQL, MySQL, MariaDB, SQL Server, SQLite, CockroachDB) - new tables, columns, keys, foreign keys, constraints or indexes, a many-to-many or membership table, money or time columns, or a migration that alters tables already holding data.
---

# Designing database schemas

## Overview

Most schema mistakes are not wrong types. They are decisions left to the database's defaults: unnamed constraints, foreign keys that delete or block by accident, a link table with the wrong identity, a migration that locks a busy table. Every schema you hand back makes those decisions explicitly.

## What a schema you hand back contains

Write the DDL so each item below is visible in it, then one short note per decision.

1. **A name on every constraint and index in the design**, as `<table>_<columns>_<suffix>`: `pkey`, `key` (unique), `fkey`, `check`, `idx`. A CHECK is named after its rule (`orders_status_check`). Stay under 63 bytes on PostgreSQL. Generated SQL may leave out a name that equals the one the database would pick; the result is the same.
2. **An `ON DELETE` on every foreign key, with its reason in the notes:**
   - `CASCADE` when the child means nothing without the parent (order items of an order).
   - `SET NULL` when the child stays valid unlinked (the column is nullable).
   - `RESTRICT` for records that must outlive a delete (orders, payments, audit); deactivate or soft-delete the parent instead. SQL Server has no `RESTRICT`: write `NO ACTION`, which blocks the delete there.
   - Never `CASCADE` across a module or bounded-context boundary.
   - An index on every foreign key column, unless an existing index starts with it.
3. **Money as `numeric(p,s)`**, scale = the currency's smallest unit (usually 2). Integer minor units only as `bigint`, and only with the currency code stored next to it. Never `double precision`, `real` or PostgreSQL `money`.
4. **The right identity for every link table:**

   | The link... | Key | Name |
   |---|---|---|
   | only pairs two rows, no data of its own | composite primary key over the two FKs | both plural: `members_roles` |
   | is referenced by other tables, or crosses a module boundary | surrogate `id` + unique constraint over the pair | singular then plural: `member_roles` |
   | has its own lifecycle (status, settings, joined/left) and inbound FKs | surrogate `id` + unique over the pair | a domain noun: `members`, `enrollments` |

   A user in several tenants is the third kind: `members (id, tenant_id, user_id, ...)` with `unique (tenant_id, user_id)`.
5. **For a migration on a table that already holds data, the locking cost of each statement**, and the safe form where it differs. On PostgreSQL:
   - Indexes: `CREATE INDEX CONCURRENTLY`, outside a transaction block, one per statement.
   - A new NOT NULL column with a per-row value: add it nullable, backfill in batches, then `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE CONSTRAINT`, and `SET NOT NULL` (12+ skips the scan); on 18, `NOT NULL ... NOT VALID` directly.
   - A constant default is cheap (PostgreSQL 11+); a volatile one (`random()`, `clock_timestamp()`) rewrites the table.
   - `SET LOCAL lock_timeout = '5s'` at the top, so a blocked migration fails instead of queueing every query behind it.
   - On MySQL (8.4), state `ALGORITHM` and `LOCK` so a statement that would copy the table fails instead:
     - Adding a foreign key copies the table while `foreign_key_checks` is on; turned off, it checks no existing row. Plan the copy, or check orphans yourself first.
     - Renaming a column (`RENAME COLUMN`) is instant and keeps its keys and indexes; a column a generated column reads cannot be renamed (3108) until that column is dropped.
     - Appending ENUM values at the end is instant; inserting, reordering or removing one copies the table, and removing a value rows still hold fails (1265).
     - Shortening a column, or a smaller numeric type, fails on any value that no longer fits in strict mode (the default); without it MySQL clamps or truncates silently.
     - An index a foreign key relies on cannot be dropped (1553) until another index starting with its columns exists: create the new one first.
   - On SQL Server, `ONLINE = ON` where the edition has it.
6. **A verification step.** If the Schemity MCP server is connected, stage the design with `propose_changes`, run `lint`, and on a connected diagram run `analyze_impact` (or `analyze_migration_file` for SQL you wrote). Fix each lint finding in the design and run `lint` again; a finding you keep gets one line saying why. Hand back no finding unexplained. Without the server, list the assumptions you made about row counts and engine version.
7. **Scenario SQL for the rules the schema has to enforce, run on a throwaway database.**
   - **Before the DDL**, list the rules from the brief that a constraint must enforce: a task's assignee belongs to the task's company, an order with payments cannot be deleted, one membership per user per tenant. Written from the brief, the scenarios test the rules; written from your DDL, they only repeat it.
   - For each rule, one statement that must succeed and one that must fail, with the error you expect. In a multi-tenant schema, every foreign key between two tenant tables gets a must-fail cross-tenant insert: foreign key checks bypass row-level security, so only the key refuses it. A rule about what a delete does (`CASCADE`, `SET NULL`) is a state check instead: run the delete, then assert what the child rows hold. Test each `RESTRICT` key alone: give the parent row children through that key only, or another key's refusal answers for it.
   - Make each must-fail case fail loudly when it is accepted. PostgreSQL:
     ```sql
     DO $$ BEGIN
       INSERT INTO tasks (company_id, assignee_id, title) VALUES (1, 2, 'cross-tenant');
       RAISE EXCEPTION 'FAIL: cross-tenant assignee accepted';
     EXCEPTION WHEN foreign_key_violation THEN RAISE NOTICE 'ok: cross-tenant assignee refused';
     END $$;
     ```
     A delete refused by a `RESTRICT` key raises `restrict_violation` (23001) on PostgreSQL 18 and `foreign_key_violation` (23503) on 17 and earlier; `NO ACTION` raises 23503 on both. Catch the one your key and version raise, or both. SQL Server: `BEGIN TRY <statement>; THROW 50000, 'FAIL: ...', 1; END TRY BEGIN CATCH IF ERROR_NUMBER() <> 547 THROW; END CATCH` (547 for any foreign key refusal). MySQL commits DDL on its own and has no anonymous blocks: use a scratch database you drop afterwards, run each must-fail statement as its own `mysql -e` call, and require a non-zero exit with the expected error (1452 for a child row, 1451 for a parent delete).
   - **The critical queries.** Name the queries the product runs most (a user's open tasks in a company, a project's board) and write each one. Check that it returns exactly the rows it should and never another tenant's, then that an index starts with the query's equality columns. Ask the catalog, not the planner: on a few rows every index starting with `company_id` costs about the same, so `EXPLAIN`, even with `enable_seqscan = off`, can pick a different one from run to run. PostgreSQL:
     ```sql
     -- the indexes whose leading columns are exactly cols, in any order; NULL means none
     SELECT string_agg(i.indexrelid::regclass::text, ', ')
     FROM pg_index i
     WHERE i.indrelid = 'tasks'::regclass AND i.indpred IS NULL AND i.indexprs IS NULL
       AND (SELECT array_agg(a.attname::text ORDER BY a.attname)
            FROM unnest(i.indkey[0:1]) k(attnum)   -- 0 : number of cols - 1
            JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = k.attnum)
           = ARRAY['assignee_id', 'company_id'];    -- sorted
     ```
     That proves a matching index exists, not that the planner picks it under load or that it is fast.
   - **Scale belongs in the design, not in the data.** Use as few rows as each check needs, two tenants at most. Let identity columns generate their ids and read them back with `RETURNING`; explicit ids leave the identity counter behind, and the next generated id collides. Volumes in the brief (500 companies, 50 users each) decide the design: which column leads each index, whether a count or a denormalized column earns its upkeep, partitioning. Write each such decision as a note.
   - When no constraint can hold a rule (exactly one author per reply even after the author is deleted, a parent row that must never be deleted even with no children), say so in the notes and pick on purpose: a trigger (bypassable, so say by what), a privilege such as `REVOKE DELETE` from the application role, or an accepted gap. Its scenario then tests what you built; for an accepted gap, it shows the statement is accepted and is labelled as the accepted gap. The notes say what each one does not cover.
   - Run the DDL and the scenarios on a container you started for this (`postgres:18`, `mysql:8.4`, `mcr.microsoft.com/mssql/server:2022-latest`), inside `BEGIN ... ROLLBACK` where the engine allows it so the schema stays and the test rows go, with `psql -v ON_ERROR_STOP=1` or `sqlcmd -b`, so the first failure stops the run. Never on a shared, staging or production database. Without Docker, hand the script back marked unrun, for the user to run in their sandbox.
   - Report each scenario as passed or failed. A must-fail statement that succeeds is a missing constraint: fix the design and run everything again. You wrote the schema and the scenarios, so the user signs off on both before anything ships.

## Quick reference

The rest of the defaults, one line each, with the measured reasoning behind them: [reference.md](reference.md).

## Common mistakes

| Mistake | Instead |
|---|---|
| `tenant_members` with a composite key, though members get roles and settings | `members` with its own id (rule 4) |
| `price_cents integer` | `numeric(12,2)`, or `bigint` minor units plus currency |
| FK with no `ON DELETE` "because the default is fine" | the default is `NO ACTION`; say so on purpose (rule 2) |
| `CREATE UNIQUE INDEX` on a live table inside `BEGIN ... COMMIT` | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... UNIQUE USING INDEX` |
| `description text` left nullable | `description text NOT NULL DEFAULT ''`; NULL only on a unique, foreign key, or CHECK-forbids-`''` column |
| A nullable column in a unique key that must hold for everyone | `NOT NULL`, a partial unique index, or `NULLS NOT DISTINCT` (PostgreSQL 15+) |
