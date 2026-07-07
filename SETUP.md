# Setup — What You Need Installed, and What You Don't

Every tool below was verified working on this machine (macOS, 2026-07-07) when the
workspace was built. **TL;DR: nothing needs installing to start.** This doc exists for
three situations: a fresh clone on another machine, a tool that broke, or the optional
extras the backlogs eventually reference.

---

## Required toolchain (all present and verified)

| Tool | Used by | Verified version | If missing (macOS) |
|---|---|---|---|
| **git** | everything | 2.50 (Apple) | `xcode-select --install` |
| **Node.js + npm** | TaskFlow | 26.4 / 11.17 | `brew install node` (any ≥ 20 works; the built-in `node:test` and `fetch` are what matters) |
| **Python 3** | Ledger | 3.13 | `brew install python` (≥ 3.11 works; type hints use modern syntax) |
| **JDK** | Stockroom | 24 (targets LTS 21 bytecode) | `brew install --cask temurin` (any ≥ 21) |
| **C++ compiler + make** | SearchLite | Apple clang 17, GNU Make 3.81 | both arrive with `xcode-select --install` |
| **sqlite3 CLI** | Ledger lab S2-6 | 3.51 | ships with macOS; `brew install sqlite` for newer |

Notably **not** required:

- **Maven** — Stockroom commits the Maven Wrapper (`./mvnw`); it downloads its own
  Maven into `~/.m2/wrapper/` on first run (already cached on this machine). This is
  the wrapper doing its actual job: new machine, zero build-tool setup. ADR-0001 in
  Stockroom explains the practice.
- **CMake** — SearchLite's Makefile is the verified build of record. The committed
  `CMakeLists.txt` is the industry-standard path, documented but unverified locally;
  install only if you want to exercise it: `brew install cmake`.

## Per-project first run (fresh clone)

Each README has the full version; this is the condensed map. All of these need network
on the *first* run (package downloads) and none afterward.

```bash
# Ledger — create the venv once, then everything runs from it
cd project-1-ledger-cli
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
.venv/bin/python -m pytest        # 69 tests
.venv/bin/ledger --help

# TaskFlow — one npm install
cd project-2-taskflow-web
npm install && npm test           # 48 tests
npm start                         # → http://localhost:3000

# Stockroom — the wrapper fetches Maven + deps on first run (be patient once)
cd project-3-stockroom-java
./mvnw test                       # 35 tests
./mvnw -q compile exec:java -Dexec.args="--demo"

# SearchLite — nothing to fetch at all
cd project-4-searchlite-cpp
make && make test                 # 25 tests / 94 checks
make run-demo
```

State that is deliberately NOT in git (all regenerable, all gitignored): Ledger's
`.venv/` and `*.db`, TaskFlow's `node_modules/` and `data/`, Stockroom's `target/`,
SearchLite's `bin/`/`obj/`. Deleting any of them costs one command to rebuild — see
each project's `.gitignore` for the reasoning comments.

## Known one-time quirks

- **Stockroom on JDK 24**: Maven prints `sun.misc.Unsafe` / native-access warnings
  before output. Harmless, explained in Stockroom's README; they disappear on JDK 21.
- **SearchLite sanitizer builds** (`make MODE=debug`): compile clean everywhere, but
  ASan-instrumented binaries may hang under sandboxed/automation environments. Run
  them from a normal terminal.
- **Paths contain a space** (`Coding Training`) — quote paths in your own shell
  commands and scripts.

## Optional installs, keyed to when the curriculum needs them

| Tool | Install | Needed when |
|---|---|---|
| **gh** (GitHub CLI) | `brew install gh` then `gh auth login` | Pushing the project repos to GitHub, creating repos/PRs from the terminal — the git-workflow doc's PR practice |
| **IntelliJ IDEA CE** | `brew install --cask intellij-idea-ce` | Debugging module §3 — the realistic Java debugger (nobody uses `jdb` by choice) |
| **ruff + black** | installed *into Ledger's venv* by story S2-5, not globally | Ledger tech-debt story S2-5 |
| **CMake** | `brew install cmake` | Only to exercise SearchLite's CMake path |
| **hyperfine** | `brew install hyperfine` | Debugging module §4, proper CLI benchmarking (nice-to-have; `time` suffices) |
| **snakeviz** | `pip install snakeviz` (in the venv) | Visualizing cProfile output in Lab 3 (optional) |

## Network access — what phones home and when

For a fully offline session: everything builds and runs offline *after* first-run
downloads. The exceptions, all opt-in: `npm install` / `pip install` / `./mvnw`'s
first run (package registries); the Ledger S3-1 lab (calls the free Frankfurter
exchange-rate API — that's the point of the lab); the TaskFlow S3-1 lab (GitHub's
API); and pushing to GitHub. No project code calls the network at runtime otherwise,
and no test suite touches the network at all — that's a deliberate invariant
(engineering-practices doc §2, networking doc §6).

## Fresh machine from zero (the 10-minute version)

1. `xcode-select --install` — git, clang, make.
2. Install [Homebrew](https://brew.sh), then: `brew install node python && brew install --cask temurin`.
3. Clone the handbook repo, then each project repo, into one parent directory (the
   handbook's cross-links assume siblings: `project-1-ledger-cli/`, etc.).
4. Run the per-project first-run block above.
5. Verify the world: each project's test suite green = you're exactly where this
   machine was at `v0.1.0`.
