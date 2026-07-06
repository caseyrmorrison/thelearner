# Glossary — Industry Vocabulary, Decoded

The words you'll hear in standups, PRs, and design reviews. Skim now; return when a doc
or teammate (real or AI) uses one.

---

## Process & Planning

- **Backlog** — the ordered list of work not yet started. "It's in the backlog" ranges from "next sprint" to "never," depending on tone.
- **Story / User story** — a unit of work framed as user value. See the Agile doc.
- **Epic** — a story too big for one sprint; a container for related stories.
- **Acceptance criteria (AC)** — testable conditions that define a story as done.
- **Spike** — a time-boxed research task ("spend a day evaluating SQLite vs Postgres"). Output is knowledge, not shipped code.
- **Velocity** — story points a team completes per sprint. A planning tool, not a performance metric — teams that are graded on velocity inflate estimates.
- **Grooming / refinement** — pre-planning meeting to clarify and estimate upcoming stories.
- **Standup** — daily 15-minute sync: did / will do / blocked.
- **Retro** — end-of-sprint reflection: keep / stop / start.
- **WIP limit** — cap on simultaneous in-progress items (Kanban). Finish before you start.
- **Timebox** — fixed time budget for an open-ended activity; when it expires, you stop and decide with what you have.
- **MVP** — minimum viable product; the smallest thing that tests the idea.
- **Scope creep** — a task quietly growing during execution. The main reason estimates fail.

## Code & Review

- **PR / MR** — pull request / merge request (same thing; GitHub vs GitLab dialect). A proposed change plus the conversation about it.
- **LGTM** — "looks good to me"; an approval.
- **Nit** — reviewer shorthand for "trivial style point, don't block on it."
- **Rubber-stamp** — approving without really reading. What happens to 2,000-line PRs.
- **Diff / changeset** — the actual lines changed.
- **Refactor** — restructuring code *without changing behavior*. If behavior changes, it wasn't a refactor.
- **Tech debt** — shortcuts whose cost compounds. Deliberate debt (documented, scheduled) is a tool; accidental debt is just decay.
- **Code smell** — a surface pattern that hints at a deeper design problem.
- **Bikeshedding** — debating trivia (tabs vs spaces) because the hard question is harder. From Parkinson's law of triviality.
- **Yak shaving** — the recursive side-quest chain you're on when someone asks "why are you reading TLS specs? weren't you fixing a button?"
- **Cargo-culting** — copying a practice without understanding why it exists. The ADRs in these projects exist to prevent exactly this.

## Build, Test, Ship

- **CI / CD** — continuous integration / delivery-or-deployment. Robots that build, test, and ship on every change.
- **Pipeline** — the ordered stages CI runs (lint → build → test → package).
- **Green / red build** — passing / failing CI. "Don't merge on red."
- **Flaky test** — passes sometimes, fails sometimes, changes nothing in between. Corrosive to trust; fix or delete.
- **Regression** — something that used to work and now doesn't. Regression tests pin down past bugs forever.
- **Smoke test** — the fast "is it fundamentally alive?" check after a deploy.
- **Fixture** — prepared data/objects a test runs against.
- **Mock / stub / fake** — stand-ins for real dependencies in tests. (Fake = working lightweight implementation, like an in-memory repository; mock = records calls for verification; stub = returns canned answers.)
- **Artifact** — the built output (binary, jar, image) that ships.
- **Feature flag** — runtime switch that hides unfinished features in shipped code. Enables trunk-based development.
- **Rollback / roll forward** — undo a bad deploy vs fix it with a new one.
- **Canary** — releasing to a small % of users first, like the mine bird.

## Environments & Operations

- **Local / dev / staging / prod** — the ladder of environments. Staging imitates prod; prod is the one with real users and real consequences.
- **"In prod"** — where code stops being homework and starts being responsibility.
- **Incident / outage / sev1** — production is broken; severity levels rank how badly.
- **Post-mortem** — the blameless written analysis after an incident.
- **On-call** — the rotation of who gets paged when prod breaks at 3 a.m.
- **Observability** — logs, metrics, traces: can you tell what the system is doing without ssh-ing in and guessing?
- **SLA / SLO** — external promise / internal target for reliability ("99.9% of requests under 200ms").

## Architecture & Data

- **Monolith / microservice** — one deployable vs many. See the Architecture Guide.
- **API contract** — the agreed request/response shapes between systems. Breaking it breaks strangers' code.
- **Idempotent** — safe to apply twice (same result). Why retries are safe for PUT but scary for POST.
- **Race condition** — correctness depends on timing luck. The bug class concurrency buys you.
- **Migration** — a scripted, versioned change to a database schema.
- **Serialization** — turning objects into bytes (JSON, protobuf) and back.
- **Cache invalidation** — knowing when stored copies are stale; one of the "two hard things in computer science" (with naming things, and off-by-one errors).
- **Backwards compatible** — old clients keep working after your change.

## The Social Layer

- **Bus factor** — how many people can be hit by a bus before the project is doomed. Documentation raises it.
- **Onboarding** — turning a new hire into a productive teammate. These repos' LEARNING_PATH.md files are onboarding docs for you.
- **Ship it** — the decision that done-enough beats perfect. The hardest skill on this page.
