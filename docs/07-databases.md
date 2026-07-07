# Databases — Integrating and Living With Persistent Data

The database outlives the application. Companies rewrite their apps every few years;
the data — and the schema decisions encoding it — survive every rewrite. That's why
storage decisions are architecture decisions (they're the "expensive to change later"
kind), and why this module exists.

You already have working database code in this workspace: Ledger ships a real SQLite
repository. This doc gives you the concepts around it, then a lab track that turns
concepts into reps.

---

## 1. The Mental Model

An application is a process: it starts, computes, dies. A database is the part of the
system that *remembers* — and it's shared: tomorrow's version of your code, the
analytics job, and the intern's script all read what you wrote today. Three consequences
drive everything below:

1. **The schema is a contract with the future.** Sloppy tables are tech debt with
   compound interest.
2. **The database can defend the data better than your code can** — constraints run
   even when someone bypasses your service layer.
3. **Data volume grows monotonically.** The query that was instant at 1k rows is the
   outage at 10M. (See the algorithms doc's "what is n in production?")

## 2. Schema Design — Modeling in Tables

- **Normalization**, practically: every fact lives in exactly one place. If a customer's
  name is stored on every order row, renaming the customer means updating n rows and
  missing one means inconsistency. Split into `customers` + `orders` with a foreign key.
  You studied the normal forms; in industry the working rule is: **normalize until it
  hurts, denormalize (deliberately, measurably) until it works.** Denormalization is a
  *performance* decision that trades write-anomaly risk for read speed — make it late,
  and write it down (ADR).
- **Keys:** every table gets a primary key. Surrogate keys (auto-increment id / UUID)
  vs natural keys (email, SKU) is a classic team debate — natural keys look clean until
  the "never changes" assumption breaks (emails change; countries split). Research both;
  default surrogate.
- **Constraints are invariants, enforced at the last line of defense:** `NOT NULL`,
  `UNIQUE`, `FOREIGN KEY`, `CHECK (amount_cents >= 0)`. Ledger's service layer validates
  input, *and* the schema should refuse garbage anyway — defense in depth, because some
  future writer won't go through your service. Compare: TaskFlow's "graph is always a
  DAG" invariant can't be expressed as a SQL constraint, so it's enforced in the service
  — knowing *which* invariants the DB can hold for you is the skill.
- **Types matter:** Ledger stores money as INTEGER cents (ADR-0002) because SQLite would
  happily store floats and corrupt totals silently. Dates as ISO-8601 TEXT in SQLite
  (sorts correctly); know your engine's real types vs its affinities.

## 3. Transactions — All or Nothing

A transaction makes a group of statements atomic: transfer = debit + credit, and a crash
between them must not lose money. **ACID** in working terms:

- **Atomic** — the group commits or rolls back as one.
- **Consistent** — constraints hold before and after.
- **Isolated** — concurrent transactions don't see each other's half-done work. Levels
  (read committed → serializable) trade correctness for throughput; research "isolation
  anomalies" when you first meet a concurrency bug, not before.
- **Durable** — committed means on disk, surviving power loss.

Practical rules: wrap multi-statement changes in explicit transactions; keep
transactions short (they hold locks); never do network calls inside one. Ledger's
CSV import is the natural place to feel this — see Lab 4.

## 4. Indexes — The B-tree Earns Its Tuition

You know B-trees from class; here's the working version. Without an index, `WHERE
category = 'groceries'` scans every row — O(n) per query. An index is a B-tree keyed on
the column: O(log n) lookups, at the price of extra space and extra work on every
INSERT/UPDATE (each index must be maintained). Hence:

- Index what you *filter and join on*, not everything. Write-heavy tables pay per index.
- **Composite indexes** serve their columns left-to-right: an index on `(month,
  category)` accelerates `WHERE month = ?` and `WHERE month = ? AND category = ?`, but
  not `WHERE category = ?` alone. Order by selectivity and access pattern.
- The database's query planner decides whether to use an index. Ask it:
  `EXPLAIN QUERY PLAN <query>` in SQLite (`EXPLAIN ANALYZE` in Postgres). "SCAN" = full
  scan; "SEARCH ... USING INDEX" = what you wanted. Reading query plans is a
  distinguishing senior-engineer skill and the core of Lab 2.

## 5. Query Safety & the Classic Performance Traps

- **Parameterized queries, always.** String-concatenated SQL is how injection happens
  (`'; DROP TABLE students;--`). Ledger's repository does this right — study it. Non-
  negotiable in review.
- **The N+1 problem:** fetch 100 orders (1 query), then loop fetching each order's
  customer (100 queries). It's invisible in dev (n=5) and murders production. Fix: JOIN
  or batched `IN (...)` queries. ORMs make N+1 *easier to write* — the main reason
  "ORM-generated SQL" needs occasional auditing.
- **Connection cost:** opening a DB connection is expensive; servers use **connection
  pools** (open k connections, lease them per request). SQLite is embedded so Ledger
  sidesteps this — but it's the first thing you'll configure against Postgres/MySQL.
  Research: "connection pool sizing" (more is not better).

## 6. Migrations — Schema as Versioned Code

The schema changes over the product's life, and "everyone manually runs ALTER TABLE" is
how environments drift apart. A **migration** is a numbered, committed script (`0003_add
_source_column.sql`) applied in order; the database records which migrations have run
(a `schema_version` table). Rules teams live by:

- Migrations are immutable once merged — fix mistakes with a new migration.
- Migrations run in CI against a fresh DB, so a broken one can't reach prod.
- Big-table migrations need care (locking); research "online schema change" someday.

Ledger currently uses `CREATE TABLE IF NOT EXISTS` — fine for Sprint 1, and its icebox
already flags the day that stops working (the first `ALTER`). Tooling: Alembic (Python),
Flyway/Liquibase (Java), node-pg-migrate, or ~50 lines of hand-rolled runner — Lab 4.

## 7. Raw SQL vs Query Builders vs ORMs

The perennial debate, honestly:

- **Raw SQL** (what Ledger does): you see exactly what runs; no magic; verbose mapping
  code. Best for learning, small apps, and performance-critical paths.
- **Query builders** (Knex, jOOQ): SQL semantics with composability and type help.
- **ORMs** (SQLAlchemy, Hibernate, Prisma): objects in, objects out — fast to start,
  and the mapping layer's leaks (N+1, surprise queries, session lifetimes) are a
  genuine cost. Teams succeed with ORMs when someone on the team can read the generated
  SQL.

The repository pattern (used everywhere in this workspace) is what keeps this choice
*contained*: swap raw SQL for an ORM and the service layer shouldn't notice. That
containment is worth more than the choice itself.

## 8. Beyond Relational — the 10-Minute NoSQL Map

- **Key-value** (Redis): O(1) lookups by key, usually in-memory; caching, sessions.
- **Document** (MongoDB): JSON blobs, flexible schema — which really means *the schema
  moves into your application code*, for better and worse.
- **Wide-column** (Cassandra): write-heavy scale-out, query patterns fixed up front.
- **Graph** (Neo4j): when relationships *are* the data (TaskFlow's dependency graph
  would qualify at absurd scale).

Default professional advice: **use a relational DB (usually Postgres) until you have a
measured reason not to.** Most "we need NoSQL" moments are actually "we need an index."

## 9. Lab Track — Where to Practice, In Order

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Read `src/ledger/repository.py` end to end with its LEARNING_PATH questions | Ledger | Repository pattern, parameterized SQL, cents-at-rest |
| 2 | **Story S2-6 (specced in Ledger's backlog):** open your ledger.db with the `sqlite3` CLI, run `.schema`, run `EXPLAIN QUERY PLAN` on the list/report queries, add an index, measure again | Ledger | Query plans, indexes, measuring instead of guessing |
| 3 | **Story S2-1 in TaskFlow:** implement `sqliteTaskRepository.js` on the open feature branch — the swap must touch zero service/route code | TaskFlow | Schema design from scratch, repository substitutability |
| 4 | Ledger icebox → real: hand-roll a migration runner (`schema_version` + numbered scripts), then use it for S2-1's `recurring_rules` table; wrap CSV import in a transaction | Ledger | Migrations, transactions |
| 5 | Stockroom icebox: JDBC + SQLite `ProductRepository` | Stockroom | The same pattern in Java's ecosystem (DriverManager, try-with-resources) |

**Research prompts:** how SQLite differs from Postgres (embedded vs server, typing,
concurrency model — SQLite's single-writer lock); what WAL mode is; covering indexes;
why `SELECT *` is banned in many codebases; optimistic vs pessimistic locking.
