# Coding Training — A Simulated Software Organization

Welcome aboard. This directory is structured like a small software company with four
product teams. You've just been hired onto all four of them.

You already have the theory (CS undergrad + CE masters). What this workspace teaches is
everything *around* the code that school skips: how real projects are structured, how
teams decide what to build, how architecture decisions get made and recorded, how git is
actually used day-to-day, and how work flows through sprints from idea to shipped feature.

---

## The Four Projects

Each project is an independent git repository — just like separate repos in a real
company's GitHub organization. Each one is a *working, tested starting point*, not a
finished product. The backlog of what to build next is part of the deliverable.

Each project name links to its GitHub repository; the local working directory is noted
alongside.

| # | Project | Stack | What it teaches | Difficulty |
|---|---------|-------|-----------------|------------|
| 1 | [Ledger](https://github.com/caseyrmorrison/thelearner-ledger) — personal finance CLI (`project-1-ledger-cli/`) | Python 3.13, SQLite, pytest | Layered architecture, CLI design, correct money math (`Decimal`), packaging, repository pattern | ⭐ Start here |
| 2 | [TaskFlow](https://github.com/caseyrmorrison/thelearner-taskflow-web) — team sprint board (`project-2-taskflow-web/`) | Node/Express + vanilla JS SPA | Full-stack client/server, REST API design, HTTP, topological sort for task dependencies, frontend without frameworks first | ⭐⭐ |
| 3 | [Stockroom](https://github.com/caseyrmorrison/thelearner-stockroom) — inventory & orders (`project-3-stockroom-java/`) | Modern Java (records, sealed types, streams), Maven, JUnit 5 | OOP design patterns (strategy, factory, repository), domain modeling, priority queues, build tooling | ⭐⭐⭐ |
| 4 | [SearchLite](https://github.com/caseyrmorrison/thelearner-searchlite) — mini search engine (`project-4-searchlite-cpp/`) | C++17, Make/CMake | Systems programming, RAII, move semantics, inverted indexes, TF-IDF ranking, manual memory discipline | ⭐⭐⭐⭐ |
| 5 | [TaskFlow iOS](https://github.com/caseyrmorrison/thelearner-taskflow-ios) — mobile client of #2 (`project-5-taskflow-ios/`) | Swift 6, SwiftUI | The iOS platform, value types, structured concurrency, declarative UI, MVVM, app lifecycle | ⭐⭐⭐ |
| 6 | [TaskFlow Android](https://github.com/caseyrmorrison/thelearner-taskflow-android) — mobile client of #2 (`project-6-taskflow-android/`) | Kotlin, Jetpack Compose | The Android platform, coroutines/Flow, recomposition, build-enforced layering, Gradle | ⭐⭐⭐ |
| 7 | [Rebound](https://github.com/caseyrmorrison/thelearner-rebound) — Breakout clone, no engine (`project-7-rebound-game/`) | JavaScript, HTML5 canvas | Game loops & fixed timestep, collision math, state machines, game feel, when to graduate to an engine | ⭐⭐ |
| 8 | [Switchboard](https://github.com/caseyrmorrison/thelearner-switchboard) — networked key-value store (`project-8-switchboard/`) | Python 3.13, raw TCP sockets, pytest | Network programming below HTTP: sockets, message framing, wire-protocol design, concurrent-server models | ⭐⭐⭐ |
| 9 | [Nabla](https://github.com/caseyrmorrison/thelearner-nabla) — autodiff & neural net from scratch (`project-9-nabla/`) | Python 3.13, stdlib only, pytest | The math of ML made code: reverse-mode automatic differentiation, backpropagation, gradient descent, training a real net — no NumPy/PyTorch | ⭐⭐⭐⭐ |
| 10 | [Tally](https://github.com/caseyrmorrison/thelearner-tally) — double-entry ledger engine (`project-10-tally/`) | Python 3.13, stdlib only, pytest | The foundation of fintech: double-entry bookkeeping, balanced journal entries, trial balance, money conservation as an invariant, idempotent append-only posting | ⭐⭐⭐ |
| 11 | [Aegis](https://github.com/caseyrmorrison/thelearner-aegis) — defensive crypto/auth primitives (`project-11-aegis/`) | Python 3.13, stdlib only, pytest | Security engineering: salted password hashing, HMAC, constant-time comparison, TOTP 2FA, CSPRNG tokens — verified against RFC vectors, with insecure-vs-secure contrasts | ⭐⭐⭐ |
| 12 | [Depot](https://github.com/caseyrmorrison/thelearner-depot) — Steam install & Workshop locator (`project-12-depot/`) | Python 3.13, stdlib only, pytest | File-format literacy and Steam modding recon: parsing Valve's VDF/ACF manifests to find installed games, App IDs, install paths, and subscribed Workshop mods | ⭐⭐ |

**Recommended order:** 1 → 2 → 3 → 4. Each project assumes slightly more independence
than the last. But they're self-contained — follow your interest if something pulls you.
The two mobile projects (5, 6) are clients of TaskFlow's REST API — do project 2 first,
then either or both; building the same app twice is the point (the architecture stays
constant while the platform changes). Project 8 (Switchboard) goes *below* the HTTP that
projects 2, 5, and 6 assume — build it after project 2 (and after reading handbook module
08) to see what a web framework hides: raw sockets, framing, and protocol design.

**Tooling:** everything runs with a standard dev setup (git, Node, Python 3, a JDK,
clang/make) — see [SETUP.md](SETUP.md) for the verified versions, per-project first-run
commands, and the short list of optional extras.

**Published on GitHub.** Each project is its own public repository (the multi-repo
pattern this workspace teaches) — linked from the table above, and each carries its full
teaching git history, its `v0.1.0` tag, and its open `feature/s2-*` (and `exercise/*`)
branches. This handbook lives in
[caseyrmorrison/thelearner](https://github.com/caseyrmorrison/thelearner).

## The Engineering Handbook

Read these before (or alongside) the first project. They're written the way an
onboarding wiki at a good company would be:

1. [Agile & Working on a Team](docs/01-agile-and-teamwork.md) — Scrum, Kanban, user stories, sprints, and how to simulate a team solo
2. [Git Workflow](docs/02-git-workflow.md) — branching strategies, conventional commits, pull requests, code review, releases
3. [Engineering Practices](docs/03-engineering-practices.md) — SOLID, testing, CI/CD, code review checklists, documentation culture
4. [Architecture Guide](docs/04-architecture-guide.md) — layered architecture, the repository pattern, REST design, ADRs, and which project demonstrates what
5. [Algorithms in Practice](docs/05-algorithms-in-practice.md) — the algorithms these projects use, and where classic CS actually shows up at work
6. [Databases](docs/07-databases.md) — schema design, transactions, indexes, migrations, and the ORM debate — with a hands-on lab track through the projects
7. [Networking & API Integration](docs/08-networking-and-apis.md) — HTTP anatomy, consuming external APIs safely (timeouts, retries, auth, rate limits), webhooks — with its own lab track
8. [Debugging & Observability](docs/09-debugging-and-observability.md) — the debugging method, per-stack debugger fluency, profiling, logging, and a planted **bug hunt** to practice on
9. [iOS Development](docs/11-ios-development.md) — the platform model, Swift for polyglots, SwiftUI's state→render loop, MVVM, App Store reality
10. [Android Development](docs/12-android-development.md) — components & lifecycle, Kotlin as modern-Java's destination, Compose, Gradle, Play reality
11. [Game Development](docs/13-game-development.md) — the game loop, fixed timesteps, collision detection, "juice", and an honest Godot/Unity/Unreal decision map
12. [Anatomy of a Codebase](docs/14-anatomy-of-a-codebase.md) — why each ecosystem's projects are shaped the way they are: tooling contracts vs. conventions, what's committed vs. ignored, and how to read any unfamiliar repo (each project's ARCHITECTURE.md has a matching "Why this shape" section)
13. [Game Modding](docs/15-game-modding.md) — how to mod a game you didn't write: the moddability spectrum, reconnaissance, file formats & mod loaders, the reverse-engineering deep end, and the legal/ethical rules — with a "make Rebound moddable" lab
14. [Network Programming](docs/16-network-programming.md) — the layer beneath HTTP: sockets, why TCP is a byte stream (message framing), wire-protocol design, and the concurrent-server models (thread-per-connection vs. event loop) — anchored by the Switchboard project
15. [Machine Learning](docs/17-machine-learning.md) — a math-forward refresher: the learning problem as optimization, where losses come from (maximum likelihood), gradient descent, backpropagation derived, and attention/transformers — anchored by the Nabla project (equations render on GitHub)
16. [FinTech](docs/18-fintech.md) — the field from zero, through an engineer's lens: money as a ledger (double-entry bookkeeping), how payments actually move, banking/compliance/lending, and why fintech is hard mode for engineers — anchored by the Tally project
17. [Cybersecurity](docs/19-cybersecurity.md) — building systems that resist attack, defensively: the security mindset, threat modeling, the vulnerability classes and their defenses, cryptography for engineers, auth, secrets & supply chain, and the ethics/law of the field — anchored by the Aegis project
18. [Modding Steam Games](docs/20-steam-game-modding.md) — the practical Steam layer on top of the game-modding module: the Workshop, finding a game's files, the `.acf`/`.vdf` manifest formats, publishing to the Workshop, and the platform gotchas (integrity-verify, auto-updates, VAC bans) — anchored by the Depot project
19. [Glossary](docs/06-glossary.md) — industry vocabulary, decoded

## The Capstone

When you've done a Sprint 2 in both Stockroom and Ledger, the
[**Procurement Pipeline capstone**](docs/10-capstone-procurement-pipeline.md) connects
them: Stockroom exposes its first real HTTP API, Ledger grows a `purchase` command that
orders through it, and you — owning both sides — run the full lifecycle of a real
integration: contract-first design, implementation on each side, failure drills, and
surviving a breaking change. It's the closest thing here to genuine industry experience.

## How to Work Through a Project

Every project repo has the same skeleton. Learn it once, use it everywhere:

```
project-x/
├── README.md              ← elevator pitch, quick start, repo tour
├── CONTRIBUTING.md        ← team conventions: branches, commits, review, definition of done
├── docs/
│   ├── ARCHITECTURE.md    ← how the system is shaped, and why
│   ├── SPRINT_BACKLOG.md  ← product vision, epics, user stories — YOUR to-do list
│   ├── LEARNING_PATH.md   ← ordered reading path through the code + research prompts
│   └── adr/               ← Architecture Decision Records (numbered, immutable)
├── .github/workflows/     ← CI pipeline definition (annotated)
└── src/ + tests/          ← heavily commented working code
```

The workflow, per project:

1. **Onboard.** Read the README, then ARCHITECTURE.md, then follow LEARNING_PATH.md
   through the source. The code comments explain *why*, not *what* — design reasoning,
   alternatives that were rejected, and trade-offs a team would debate.
2. **Run it.** Build it, run the tests, use the app. Break something on purpose and watch
   a test fail. Trust nothing you haven't executed.
3. **Read the git history.** `git log --oneline --graph --all` shows how the project was
   built up commit by commit — each commit is a lesson in how much to change at once.
4. **Pick up a story.** Open SPRINT_BACKLOG.md. Sprint 1 is done (that's the code you're
   reading). Sprint 2 stories are specced with acceptance criteria and research pointers.
   Each repo already has an open `feature/s2-*` branch with a stubbed starting point —
   check it out and finish it.
5. **Work like a team of one.** Branch → commit incrementally with tests → self-review
   the diff before merging → merge → retro (see the Agile doc for a solo-retro format).

## Simulating the Team

You don't have coworkers here, so the docs play their roles:

- **Product owner** → SPRINT_BACKLOG.md tells you what and why, never how.
- **Tech lead** → ARCHITECTURE.md and the ADRs constrain your design choices — respect
  them or write a new ADR superseding the old one (that's how it works in real life).
- **Code reviewer** → CONTRIBUTING.md has the review checklist. Review your own PRs
  against it — or ask an AI assistant to review your diff and argue with its feedback.
- **The previous engineer** → the code comments and git history.

An AI assistant (like the one that generated this scaffolding) makes a good standin for
pairing, code review, and design debate. Good prompts: *"Review this diff like a senior
engineer,"* *"Argue against my approach to story S2-3,"* *"Interview me about this
project's architecture."*

## Why the Repos Are Organized This Way

This root directory is itself a git repo, but it only tracks the handbook (`docs/` and
this README). The four project directories are each their own independent repos, and the
root's `.gitignore` excludes them. Two reasons:

1. **Realism.** Companies keep a shared engineering handbook/wiki plus one repo per
   service or product. Monorepos exist too (Google, Meta) — see the Architecture Guide
   for that trade-off — but multi-repo is what you'll meet most often.
2. **Mechanics.** Git repos don't nest cleanly. A repo inside another repo becomes an
   un-tracked "gitlink" unless you use submodules — a feature with enough sharp edges
   that many teams ban it. Ignoring the nested repos sidesteps all of it.
