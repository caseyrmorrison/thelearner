# Architecture Guide — How Systems Get Their Shape

Architecture is the set of decisions that are expensive to change later. This guide
covers the patterns used across the four projects and the reasoning tools behind them.

---

## 1. How to Think About Architecture

Three questions drive almost every architectural choice:

1. **What will change?** Isolate the volatile parts behind interfaces (storage engines,
   UI frameworks, pricing rules). Stable things can be coupled cheaply.
2. **What must not break?** Correctness of money math, of user data, of ordering
   guarantees — the invariants. Architecture protects invariants.
3. **What can we afford?** Every layer of flexibility has a complexity price. Junior
   architects add abstraction; senior architects delete it. YAGNI applies to
   architecture doubly.

The projects deliberately choose *boring, simple* architectures and record the reasoning
in ADRs. Boring is a feature: novelty budget is real (research: "choose boring
technology" — Dan McKinley).

## 2. Layered Architecture (used by all four projects)

The workhorse pattern of business software. Dependencies point strictly downward:

```
┌─────────────────────────────┐
│ Presentation                │  HTTP routes / CLI parsing / UI
│  "translate the outside     │  knows: HTTP codes, argv, JSON
│   world to and from calls"  │  never: business rules, SQL
├─────────────────────────────┤
│ Service / Domain            │  the actual rules of the business
│  "the reason the app exists"│  knows: what a valid task/order/txn is
│                             │  never: HTTP, SQL, file paths
├─────────────────────────────┤
│ Persistence (Repository)    │  storage mechanics
│  "keep things, find things" │  knows: SQL / file formats
│                             │  never: business rules
└─────────────────────────────┘
```

Why it earns its keep:

- **Testability** — the service layer tests without a web server or database.
- **Substitutability** — TaskFlow's JSON-file store swaps for SQLite without touching
  business logic (that's literally a Sprint 2 story).
- **Navigability** — a new teammate knows where any given code *should* live.

The failure mode: **anemic layers**, where the service layer is a pass-through and logic
leaks into routes or SQL. If a route handler contains an `if` about business rules, it's
in the wrong layer.

## 3. The Repository Pattern (the recurring theme)

Every project defines an interface like `TaskRepository` / `TransactionRepository` with
methods in *domain* terms (`findByStatus`, `save`) — never storage terms — plus one or
more implementations (in-memory, JSON file, SQLite).

What this buys, concretely:

- Tests inject an in-memory fake: fast, deterministic, no cleanup.
- Storage can be upgraded (file → SQLite → Postgres) as a contained change.
- The domain layer stays pure — it can't even *express* a dependency on SQL.

This is **Dependency Inversion** made concrete, and a small taste of
**Hexagonal Architecture / Ports & Adapters** — the generalization where *every*
external thing (DB, HTTP, clock, email) sits behind a domain-owned interface ("port")
with swappable "adapters." Research it after you've felt the repository version work.

## 4. Client–Server & REST (TaskFlow)

Full-stack means two programs talking over HTTP:

- The **API contract** is the real interface between frontend and backend teams. Design
  it first; document it (TaskFlow's README does); version it when it must break.
- **REST conventions**: nouns for resources (`/api/tasks/42`), HTTP verbs for actions
  (GET read / POST create / PUT-PATCH update / DELETE), status codes as vocabulary
  (200/201/204, 400 bad input, 404 missing, 409/422 conflict or unprocessable, 500 our
  fault). Consistent, guessable APIs reduce coordination cost between teams.
- **Statelessness**: each request carries what's needed; the server holds no per-client
  session memory. This is what lets you run 40 identical servers behind a load balancer.
- Research after building: idempotency (why PUT can be retried but POST can't),
  pagination strategies, API versioning, OpenAPI/Swagger.

## 5. Monolith vs Microservices (and monorepo vs multi-repo)

- All four projects are **monoliths**: one deployable unit. For nearly every new
  product this is correct — services add network failure modes, distributed data
  consistency, and operational cost that only pay off at organizational scale
  ("microservices solve an org-chart problem, not a technical one").
- The honest middle path: build a **modular monolith** (clean internal boundaries — the
  layering above) so that *if* a piece needs to become a service later, the seam already
  exists.
- Multi-repo vs monorepo is the *source layout* version of the question. This workspace
  is multi-repo (one repo per project) because that's the common case; monorepos
  (Google, Meta) trade tooling investment for atomic cross-project changes.

## 6. Architecture Decision Records (ADRs)

The single highest-leverage documentation habit. One short numbered file per decision:

```markdown
# ADR-0002: Store tasks in a JSON file behind a repository interface

## Status    Accepted (2026-07)
## Context   Sprint 1 needs persistence. Candidates: JSON file, SQLite, Postgres.
             One developer, no ops budget, data volume is tiny.
## Decision  JSON file, hidden behind TaskRepository.
## Consequences
  + Zero dependencies; storage is human-inspectable while learning.
  + Repository interface means upgrading storage is a contained change (planned S2).
  - No concurrent-write safety; fine for one user, unacceptable beyond that.
  - Full rewrite on every save: O(n) per write. Acceptable at expected n.
```

Rules: ADRs are **immutable** — wrong later ≠ wrong then. A new decision gets a new ADR
marked "Supersedes ADR-000X." The `docs/adr/` folder in each project is its team's
decision memory. **When you make a significant change in Sprint 2+, write the ADR.**
That habit alone will put you ahead of most working engineers.

## 7. Cross-Cutting Concerns You'll Meet in the Code

- **Dependency injection** — pass collaborators in (constructor args), don't construct
  them inside. All four projects wire dependencies together in exactly one place
  (`main`/`app` — the "composition root"). No framework needed; DI is a *pattern*,
  Spring/Guice just automate it.
- **Configuration** — behavior that varies by environment (ports, file paths) comes
  from env vars/config files, never hardcoded. Research: "12-factor app."
- **The composition root** — grep for where each project constructs its objects. It's
  the map of the whole system in ten lines.

## 8. Which Project Demonstrates What

| Concept | Where to see it working |
|---|---|
| Layered architecture + composition root | all four — compare how each language expresses it |
| Repository pattern, swap-the-storage exercise | Ledger (SQLite repo), TaskFlow (JSON→SQLite story) |
| REST API design, statelessness, validation at the boundary | TaskFlow |
| Strategy & factory patterns, domain modeling with types | Stockroom |
| Value objects & correct money handling | Ledger (`Decimal`), Stockroom (`Money` record) |
| Performance-conscious design, cache locality, move semantics | SearchLite |
| ADRs as decision memory | every `docs/adr/` — read all of them; they cross-reference |
