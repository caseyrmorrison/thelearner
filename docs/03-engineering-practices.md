# Engineering Practices — Writing Code a Team Can Live With

School grades code on "does it work." Teams grade it on "can someone else change it
safely in six months." These are the practices that close that gap.

---

## 1. Design Principles (and how seriously to take them)

### SOLID — the classics, with honest commentary

- **S — Single Responsibility**: a module should have one reason to change. The most
  useful of the five. In these projects: routes don't touch the database, repositories
  don't know about HTTP. When a file needs editing for two unrelated reasons, split it.
- **O — Open/Closed**: extend behavior without modifying existing code. See Stockroom's
  pricing strategies — adding a discount type means adding a class, not editing a giant `if`.
- **L — Liskov Substitution**: subtypes must honor the promises of their supertype. If
  code holding a `Repository` breaks when handed a `SqliteRepository`, the abstraction is a lie.
- **I — Interface Segregation**: many small interfaces beat one fat one.
- **D — Dependency Inversion**: depend on abstractions, not concretions — the reason
  every project here defines a repository *interface* and injects an implementation.
  This is what makes the code testable.

Honest take: SOLID is a checklist for *diagnosing* discomfort, not a recipe. Over-applied,
it produces "lasagna code" — 12 layers of indirection to print a string.

### The counterweights

- **KISS** — simple beats clever. Clever code is expensive to everyone, including future you.
- **YAGNI** — "You Aren't Gonna Need It." Don't build for imagined future requirements.
  The ADRs in these projects repeatedly choose the boring option on purpose.
- **DRY, carefully** — duplication is bad when the copies must stay in sync, *fine* when
  they'll evolve independently. "A little duplication is cheaper than the wrong
  abstraction" (Sandi Metz — worth researching).

## 2. Testing

### The pyramid

```
        /  E2E  \        few — slow, brittle, high confidence per test
       / integr. \       some — modules together (API + real storage)
      /   unit    \      many — one unit, fast, run on every save
```

- **Unit tests** dominate because they're fast enough to run constantly and they
  pinpoint failures. Every project here ships with a unit suite — run it, break code on
  purpose, watch it catch you.
- **Integration tests** catch what units can't: wiring, serialization, SQL that only
  fails against a real database. TaskFlow tests its API this way.
- **E2E** (drive the real UI) is expensive; keep a handful for critical paths.

### Habits that matter more than the taxonomy

- **Arrange–Act–Assert**: every test has three visible sections. One behavior per test.
- **Name tests as claims**: `rejects_dependency_cycle`, not `test3`. A failing test name
  should read as a bug report.
- **Test behavior, not implementation.** If a refactor (same behavior, new internals)
  breaks 40 tests, the tests were coupled to internals and are now a liability.
- **TDD** (write the failing test first) — try it on one Sprint-2 story to feel the
  loop: red → green → refactor. Many strong engineers use it selectively; almost all
  keep the *habit* of writing the test in the same sitting as the code.
- **Coverage** is a smoke detector, not a goal. 100%-covered code can still be wrong;
  chasing the number produces assert-free tests. Use it to find *untested regions*.

## 3. Code Quality Mechanics

- **Naming is design.** If you can't name a function honestly (`processData` is a
  confession), its responsibilities are wrong. Rename ruthlessly.
- **Functions**: small-ish, one job, few parameters. Deep nesting → extract or invert
  with early returns (guard clauses — used all over these projects).
- **Comments explain WHY, never WHAT.** `i++ // increment i` is noise. "Kahn's algorithm
  chosen over DFS because we also need to report *which* nodes form the cycle" is
  engineering. The scaffolds model this style deliberately.
- **Code smells** to develop a nose for: long parameter lists, feature envy (a method
  that mostly touches another class's data), shotgun surgery (one change → edits in 9
  files), god objects, boolean parameters (`render(true, false)` — of what?!).
- **Linters & formatters** end style debates by automation: Prettier/ESLint (JS),
  ruff/black (Python), Checkstyle/Spotless (Java), clang-format/clang-tidy (C++). Teams
  adopt them so review can focus on substance. Each project's CONTRIBUTING.md names its
  tooling; installing and running them is an early backlog story.

## 4. Error Handling & Logging

- Decide *where* errors are handled: at the boundary (HTTP handler, CLI main), not
  scattered mid-stack. Inner layers throw/return domain errors; the boundary translates
  them to exit codes / HTTP statuses. All four projects model this.
- Never swallow exceptions silently. An empty `catch` is a time bomb with no timer.
- **Exceptions vs error-returns** is a language-culture question: Python/Java lean
  exceptions; modern C++ increasingly likes `std::optional`/`expected`; Go returns
  errors. SearchLite discusses this in an ADR.
- Logs are for *future you at 3 a.m.*: log decisions and state transitions with context
  ("rejected order 123: insufficient stock for SKU-9"), not play-by-play noise. Learn
  log levels (debug/info/warn/error) and structured logging (key=value or JSON).

## 5. CI/CD — Automation as Team Immune System

- **Continuous Integration**: every push/PR triggers a server that builds and tests from
  scratch. Kills "works on my machine." A red build is the team's top priority.
- **Continuous Delivery/Deployment**: green main is automatically shippable (delivery)
  or automatically shipped (deployment).
- Each project contains `.github/workflows/ci.yml` — annotated GitHub Actions configs.
  They don't run locally (no GitHub remote yet), but read them: they're a precise,
  executable description of "what this team checks before code merges." Pushing a
  project to GitHub and watching CI go green is a backlog story in each repo — do it at
  least once.
- Typical pipeline stages: lint → build → unit tests → integration tests → package.
  Fail fast: cheap checks first.

## 6. Documentation Culture

What actually gets written at healthy companies, in order of importance:

1. **README** — what is this, how do I run it, where do I look next. The front door.
2. **ADRs** (Architecture Decision Records) — one short file per significant decision:
   context, options, decision, consequences. Immutable; superseded by later ADRs rather
   than edited. The answer to "why on earth is it built this way?" three years later.
   Every project here has several — read them, then write one for your first real change.
3. **CONTRIBUTING** — team conventions, mechanized.
4. **Inline docs** — docstrings/Javadoc on public APIs; WHY-comments at decision points.

Anti-pattern: the 40-page design doc nobody updates. Docs near the code, in the repo,
reviewed with the code, stay alive.

## 7. Security Hygiene (the minimum every engineer owes the team)

- **Never commit secrets.** Not passwords, not API keys, not "just for testing." Git
  history is forever (rewriting it after a leak is an incident, not a fix). Use env
  vars / `.env` files that are gitignored — TaskFlow demonstrates the pattern.
- **Validate at trust boundaries.** Anything from a user, a file, or the network is
  hostile until validated. Parameterized SQL only (Ledger shows why — research "SQL
  injection" and "little Bobby Tables").
- **Dependencies are attack surface.** `npm audit`, `pip-audit` exist; lockfiles pin
  what you actually ship. Research "supply chain attack" for why this got serious.
- Principle of least privilege — for file permissions, DB users, API tokens alike.

## 8. Definition of Done (the practices, condensed)

Before you call any story finished, in any of these projects:

- [ ] Tests written with the change, all green, run from a clean checkout
- [ ] Self-reviewed diff (`git diff main...`) against CONTRIBUTING.md
- [ ] No new warnings from the compiler/linter
- [ ] Docs touched if behavior or a decision changed (README/ARCHITECTURE/new ADR)
- [ ] You ran the actual program and used the feature like a user would
