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

| # | Project | Stack | What it teaches | Difficulty |
|---|---------|-------|-----------------|------------|
| 1 | [Ledger](project-1-ledger-cli/) — personal finance CLI | Python 3.13, SQLite, pytest | Layered architecture, CLI design, correct money math (`Decimal`), packaging, repository pattern | ⭐ Start here |
| 2 | [TaskFlow](project-2-taskflow-web/) — team sprint board | Node/Express + vanilla JS SPA | Full-stack client/server, REST API design, HTTP, topological sort for task dependencies, frontend without frameworks first | ⭐⭐ |
| 3 | [Stockroom](project-3-stockroom-java/) — inventory & orders | Modern Java (records, sealed types, streams), Maven, JUnit 5 | OOP design patterns (strategy, factory, repository), domain modeling, priority queues, build tooling | ⭐⭐⭐ |
| 4 | [SearchLite](project-4-searchlite-cpp/) — mini search engine | C++17, Make/CMake | Systems programming, RAII, move semantics, inverted indexes, TF-IDF ranking, manual memory discipline | ⭐⭐⭐⭐ |

**Recommended order:** 1 → 2 → 3 → 4. Each project assumes slightly more independence
than the last. But they're self-contained — follow your interest if something pulls you.

## The Engineering Handbook

Read these before (or alongside) the first project. They're written the way an
onboarding wiki at a good company would be:

1. [Agile & Working on a Team](docs/01-agile-and-teamwork.md) — Scrum, Kanban, user stories, sprints, and how to simulate a team solo
2. [Git Workflow](docs/02-git-workflow.md) — branching strategies, conventional commits, pull requests, code review, releases
3. [Engineering Practices](docs/03-engineering-practices.md) — SOLID, testing, CI/CD, code review checklists, documentation culture
4. [Architecture Guide](docs/04-architecture-guide.md) — layered architecture, the repository pattern, REST design, ADRs, and which project demonstrates what
5. [Algorithms in Practice](docs/05-algorithms-in-practice.md) — the algorithms these projects use, and where classic CS actually shows up at work
6. [Glossary](docs/06-glossary.md) — industry vocabulary, decoded

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
