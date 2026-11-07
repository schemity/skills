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

## Quick reference

The rest of the defaults, one line each, with the measured reasoning behind them: [reference.md](reference.md).

## Common mistakes

| Mistake | Instead |
|---|---|
| `tenant_members` with a composite key, though members get roles and settings | `members` with its own id (rule 4) |
| `price_cents integer` | `numeric(12,2)`, or `bigint` minor units plus currency |
| FK with no `ON DELETE` "because the default is fine" | the default is `NO ACTION`; say so on purpose (rule 2) |
| `CREATE UNIQUE INDEX` on a live table inside `BEGIN ... COMMIT` | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... UNIQUE USING INDEX` |
| A nullable column in a unique key that must hold for everyone | `NOT NULL`, a partial unique index, or `NULLS NOT DISTINCT` (PostgreSQL 15+) |
