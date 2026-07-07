# Anatomy of a Codebase — Why Projects Are Shaped the Way They Are

Open any of the seven projects and the folder tree tells you things before you read a
line of code. That's not an accident — project layout is a *language*, and this module
teaches you to read and speak it. For each ecosystem it answers three questions the
trees alone don't: **who enforces this shape** (the build tool? the compiler? just
culture?), **what the shape communicates**, and **what breaks if you deviate**.

Each project's `docs/ARCHITECTURE.md` has a "Why this shape" section applying this to
its own tree — this module is the cross-cutting theory.

---

## 1. Three Forces Shape Every Repo

1. **Tooling contracts.** Some layouts are *load-bearing*: Maven refuses to find code
   outside `src/main/java`; Swift Package Manager mandates `Sources/<Target>/`; the
   Java compiler requires package names to match folder paths. Deviating means fighting
   your tools daily, so nobody does — which is precisely what makes these layouts
   universal and navigable.
2. **Convention over configuration.** Where tools don't *require* a shape, ecosystems
   converge on one anyway (Python's `src/` layout, Node's `routes/`+`services/`), because
   every deviation is a question a new teammate has to ask. A conventional repo is
   pre-onboarded.
3. **Architecture made physical.** The best layouts make the design *visible and
   enforceable*: TaskFlow Android's `:core` module physically cannot import Android;
   SearchLite's `include/` vs `src/` split *is* encapsulation; Rebound's `engine/` vs
   `game/` is the port-to-Godot seam. When the folder structure matches the dependency
   rules, violations look wrong in the diff before any reviewer speaks.

The failure mode of all three: a repo shaped by accretion — `utils/`, `helpers/`,
`misc/`, `new_final2/` — where structure communicates nothing and enforces less.

## 2. The Universal Skeleton (all seven projects)

Some anatomy repeats everywhere, and employers expect it on sight:

| Item | Why it's at the root |
|---|---|
| `README.md` | The front door. GitHub renders it; humans start here. |
| `CONTRIBUTING.md` | Team rules, discoverable before the first PR. |
| `docs/` + `docs/adr/` | Decisions live *with* the code they govern, reviewed in the same PRs. |
| `.github/workflows/` | CI is code; the platform requires this exact path (a tooling contract). |
| `.gitignore` | The boundary between "source of truth" and "regenerable" — see §4. |
| `.editorconfig` | Whitespace/encoding conventions enforced by editors, ending that argument. |
| Tests in a predictable place | Mirroring source (`tests/`, `src/test/`) so the test for X is always at X's coordinates. |

## 3. The Ecosystems, Compared

| | Who dictates the shape | Domain code lives | Tests live | Config of record |
|---|---|---|---|---|
| Python (Ledger) | Culture (src layout is opt-in) | `src/ledger/` | `tests/` (outside the package) | `pyproject.toml` |
| Node (TaskFlow) | Culture only — strongest-discipline ecosystem | `server/<layer>/` | `test/` | `package.json` |
| Java/Maven (Stockroom) | **Maven + the compiler** (hard contract) | `src/main/java/com/...` | `src/test/java/...` (mirrored) | `pom.xml` |
| C++ (SearchLite) | Nobody — the most convention-free ecosystem | `include/` (API) + `src/` (impl) | `tests/` | `Makefile`/`CMakeLists.txt` |
| iOS (TaskFlow iOS) | **SPM** (hard) for the package; XcodeGen spec for the app | `TaskFlowKit/Sources/` | `TaskFlowKit/Tests/` (SPM-mandated names) | `Package.swift` + `project.yml` |
| Android (TaskFlow Android) | **Gradle + AGP** (hard): modules, `src/main/`, `res/` | `core/src/main/kotlin/` | `src/test/kotlin/` per module | `settings.gradle.kts` + `libs.versions.toml` |
| Browser game (Rebound) | Nobody — the tree ships verbatim | `src/engine/` + `src/game/` | `test/` | none (that's the point — ADR-0001) |

Patterns worth noticing across the table:

- **The JVM column is strictest.** Maven invented `src/main/<lang>` twenty years ago;
  Gradle inherited it; the Java compiler adds package-path = folder-path. Result: any
  Java/Kotlin/Android developer can navigate any JVM repo blind. That predictability is
  the payoff of rigid contracts.
- **The C column is loosest.** C++ has no package manager consensus, no standard layout,
  no test-discovery convention — every C++ shop's tree is a local dialect (research:
  the "Pitchfork layout" proposal, which SearchLite approximates). The discipline that
  Maven automates, C++ teams must supply by review.
- **JavaScript sits in between**: zero enforcement, strong conventions. TaskFlow's
  `routes/ services/ repositories/` names are doing the work Java's packages do — but
  only culture holds them up. Move a business rule into a route handler and no tool
  complains; that's why the CONTRIBUTING checklist and review exist.
- **Mobile is generated-file country.** Both mobile projects treat some "project files"
  as build *products*: the `.xcodeproj` is generated from `project.yml` (committed spec,
  generated artifact ignored), and Android's `R` class, `local.properties`, and `build/`
  are all machine-made. The rule generalizes: **commit the recipe, ignore the cake.**

## 4. What Gets Committed vs Ignored — the Anatomy Rule

Every `.gitignore` in this workspace draws the same line, and it's worth stating as a
principle: version control holds *sources of truth*; everything derivable is ignored.

- Derived: `target/`, `build/`, `bin/`, `obj/`, `node_modules/`, `.venv/`, `DerivedData/`,
  generated `.xcodeproj`, compiled `.pyc`.
- Machine-specific: `local.properties`, `.env`, IDE state (`.idea/`, `xcuserdata/`).
- Deliberate exceptions prove the rule: Stockroom **commits** the Maven Wrapper JAR and
  TaskFlow Android commits the Gradle wrapper — binaries in git! — because they are the
  *recipe for reproducing the build*, not a product of it (each has an ADR).

Interview-grade question you can now answer: "why is `package-lock.json` committed but
`node_modules/` ignored?" Same principle — the lockfile is truth (exact versions
resolved), the folder is derivable from it.

## 5. Package-by-Layer vs Package-by-Feature — the Honest Debate

Every project here packages **by layer** (`domain/`, `service/`, `repository/`;
`routes/`, `services/`) because at teaching scale the layers *are* the lesson. Know the
counter-position: at scale, layer-packaging scatters one feature across five folders,
and teams switch to **feature folders** (`tasks/`, `billing/`, `search/` — each with its
own mini-layers inside). Advocates call it "screaming architecture" (Uncle Bob's term:
the tree should scream what the app *does*, not what framework pattern it uses).

The trade-off in one line: **layer-packaging optimizes for learning and enforcing the
layering; feature-packaging optimizes for changing one feature without touring the
repo.** Watch for the moment in any growing project when most PRs touch one folder per
layer — that's the signal teams use to reorganize. (Research: "screaming architecture",
"vertical slice architecture", and how Go projects and Rails apps each institutionalized
one side of this debate.)

## 6. How to Read an Unfamiliar Repo (the transferable skill)

The order working engineers actually use, and why it works on any repo shaped by the
forces above:

1. `README.md` — what is this, how does it run.
2. The config of record (`package.json`, `pyproject.toml`, `pom.xml`, `Package.swift`,
   `settings.gradle.kts`) — it names the entry points, dependencies, and scripts; it's
   the repo's passport.
3. The folder names one level deep — with this module, they now tell you the
   architecture and the ecosystem's enforcement level.
4. The composition root (find `main`) — ten lines that wire the whole object graph.
5. The tests for whatever you're changing — executable documentation of intended
   behavior.
6. `git log --oneline -20` and the ADRs — how it got this way, and why.

Practice deliberately: open a well-known open-source repo in each ecosystem (Flask,
Express itself, Spring PetClinic, LLVM's subfolders) and walk this list. The layouts
will already feel familiar — that's this module doing its job.
