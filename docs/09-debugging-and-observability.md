# Debugging & Observability — Finding Out What's Actually Happening

Debugging is the highest-leverage skill nobody's course taught properly, because it
can't be lectured — only practiced against real confusion. This module gives you the
method, the tool ladder for all four project stacks, and (most importantly) staged
crime scenes to practice on, including a planted bug hunt.

---

## 1. The Method (this is 80% of it)

Debugging is applied science: the bug is a fact, your job is the explanation.

1. **Reproduce it first.** A bug you can't trigger on demand can't be verified fixed.
   Capture the exact steps/input; script them if possible. "Sometimes it happens" means
   "I haven't found the triggering condition yet."
2. **Minimize.** Shrink input, steps, and code path until the smallest thing that still
   fails. Half of all bugs die of embarrassment during minimization.
3. **Hypothesize, then look.** State what you believe ("the service returns the right
   data, so the bug is in rendering") and check *that*, instead of staring at code
   hoping. Each check should split the search space — binary search on the causal
   chain, the same instinct as bisecting.
4. **Read the error. Actually read it.** The line number, the type, the *first* error
   (later ones are usually debris). An astonishing fraction of "weird bugs" are solved
   inside the message already on screen.
5. **One change at a time.** Change two things and learn nothing from the result.
6. **When stuck, write it down** — a running log of what you tried and saw ("rubber
   duck" in prose). For multi-day bugs this is the difference between progress and
   circling. And question your assumptions in order of "most recently touched" first:
   the bug is usually in *your* newest code, not the library, not the compiler.
   (It is never the compiler. Until the one day it is — earn that suspicion last.)

## 2. The Tool Ladder

Escalate deliberately, cheapest first:

- **Print/log debugging** — legitimate and universal. Do it well: label everything
  (`console.log('after-merge', {taskId, status})`, not naked values), and delete them
  before commit (the linter story catches stragglers).
- **The REPL** — Python and Node both let you import your real modules and poke them
  interactively. Fastest hypothesis-checker there is for pure functions.
- **A real debugger** — pays off when state evolves over time and prints would be a
  blizzard: breakpoints, step over/into, watch expressions, and *post-mortem* inspection
  of a crash. Per-stack details in §3.
- **`git bisect`** — when the question is "*when* did this break?", binary search the
  history instead of the code. Needs a fast yes/no test per commit (see the git doc §7).
- **Sanitizers & analyzers** (C++ mostly) — ASan/UBSan turn silent memory corruption
  and undefined behavior into loud, located errors. Not optional equipment in modern
  C++ work; see the SearchLite lab.

## 3. Per-Stack Quick Reference

| Stack | Interactive debugger | Post-mortem / crash | Notes |
|---|---|---|---|
| **Python** (Ledger) | Drop `breakpoint()` in the code, run normally; or `pytest --pdb` to land in the debugger at the first failure | `python -m pdb -m ledger.cli ...` then `c`; after any crash in a REPL, `import pdb; pdb.pm()` inspects the corpse | pdb commands worth knowing: `n`/`s`/`c` (over/into/continue), `p expr`, `ll` (relist), `w` (where am I), `u`/`d` (walk the stack) |
| **Node server** (TaskFlow) | `node --inspect server/server.js`, open `chrome://inspect` → your process → DevTools with full breakpoint support over the *server* code | `node --inspect-brk` pauses before line 1 — for bugs during startup | Also: `NODE_OPTIONS=--inspect npm start` when a script wraps the entry point |
| **Browser frontend** (TaskFlow) | DevTools → Sources → breakpoint in `board.js`/`app.js`; the Network tab shows every fetch; `debugger;` statement forces a pause | Console errors carry stack traces — click through | **This is the only visibility you have here: no test suite covers the frontend.** That fact is load-bearing for the bug hunt below |
| **Java** (Stockroom) | Honest answer: nobody uses `jdb` by choice — use an IDE (IntelliJ IDEA CE): set a breakpoint, debug the test or `Main` | `jstack <pid>` dumps all thread stacks of a hung JVM | Later: Java Flight Recorder (`-XX:StartFlightRecording`) is production-grade always-on profiling, free since JDK 11 |
| **C++** (SearchLite) | `lldb bin/searchlite` → `b ranker.cpp:42` → `run search idx "heap"`; `bt` for backtrace, `frame variable` to inspect | Crashes: run under lldb and it stops *at* the fault with the whole stack | ASan/UBSan: `make MODE=debug` builds with both; then just run — corruption reports itself with file:line. Valgrind is the Linux-side equivalent worth knowing exists |

## 4. Profiling — Measure, Then Optimize

The cardinal sin is optimizing by intuition; humans guess the hot spot wrong most of
the time. The loop: **measure → find the dominant cost → fix that → measure again.**

- **Wall-clock first:** `time <cmd>` answers "is it even slow?" (`hyperfine` for proper
  benchmarking with warmup/statistics — worth installing someday).
- **Python:** `python -m cProfile -s cumtime -m ledger.cli import-csv big.csv | head -30`
  — sort by cumulative time, read top-down. (`snakeviz` renders it visually.)
- **Node:** `node --cpu-prof server/server.js` writes a `.cpuprofile`; load it in
  DevTools → Performance. For quick checks, `console.time('label')`/`timeEnd` brackets.
- **C++:** on macOS, `sample <pid>` or Instruments; on Linux, `perf record`/`perf
  report`. Compile with `-O2 -g` so the profile has symbols *and* real optimizations.
- Read profiles with Amdahl's law in your head: a 10× speedup of a 4% function buys
  you 3.7%. Find the 80% frame or don't bother.

## 5. Logging in Anger (writing logs you can debug WITH)

Print debugging is for you, now; logging is for whoever is on call, later. Rules:

- **Levels mean things.** `debug` = development noise, off in prod; `info` = state
  transitions worth an audit trail ("order 123 partially fulfilled: 8/20 SKU-1003");
  `warn` = survivable surprise; `error` = someone should look. If everything is
  `info`, nothing is.
- **Structure beats prose:** `status=422 path=/api/tasks dur_ms=12 req_id=af3` greps,
  parses, and graphs; "something went wrong in tasks!" does none of those.
- **Log at boundaries and decisions**, with identifiers (which order? which request?) —
  not play-by-play inside loops.
- **A request id stitches the story together:** generate one per incoming request,
  return it in a header, attach it to every log line that request produces. When a user
  reports an error, the id in their response turns "somewhere in the logs" into `grep`.
  TaskFlow's story S3-2 (specced in its backlog) makes you build exactly this.
- Never log secrets, tokens, or full card numbers/PII. Log redaction incidents are a
  genre of post-mortem.

## 6. Observability — Beyond One Process

The production-grade version of "what is happening?", in three signals:

- **Logs** — discrete events with context (§5).
- **Metrics** — cheap numeric aggregates over time (request rate, error rate, p95
  latency, queue depth). The four "golden signals" (latency, traffic, errors,
  saturation) come from Google's SRE book — research the term.
- **Traces** — one request's journey across services with per-hop timing; the
  distributed answer to "where did the 3 seconds go?" (OpenTelemetry is the standard.)

The bar to internalize: *can you diagnose the system from its telemetry alone, without
attaching a debugger to production?* A `/healthz` endpoint (also in S3-2) is the
smallest first step — it's what load balancers and uptime checks poll.

## 7. The Labs — Practice on Staged Crime Scenes

| # | Lab | Where | What it drills |
|---|-----|-------|----------------|
| 1 | **Bug hunt** (below) | TaskFlow, branch `exercise/bughunt-1` | Symptom → cause with browser DevTools; the class of caching/invalidation bugs |
| 2 | Put `breakpoint()` inside `LedgerService.budget_status`, run `ledger budget status --month 2026-07`, and walk the stack: inspect the transactions, step into `monthly_by_category`, watch the totals dict grow | Ledger | pdb fluency; reading a layered call stack from the inside |
| 3 | Generate a 200k-row CSV (throwaway script), `cProfile` the import, name the top-3 cumulative-time functions, then decide: is anything worth fixing? Write the two-sentence answer | Ledger | Profiling; the discipline of *not* optimizing without cause |
| 4 | Rebuild SearchLite with `make clean && make MODE=debug`, rerun `make test` and `run-demo` under sanitizers; then read the Makefile to explain exactly what changed in the compile line | SearchLite | Sanitizers as standard equipment; build-flag literacy |
| 5 | `git bisect` drill: in TaskFlow, find which commit introduced the blocked badge using `git bisect run grep -q 'badge blocked' public/js/board.js` — mechanics practice on a real history | TaskFlow | Bisect fluency before you need it in anger |

### Lab 1: the bug hunt, full briefing

A user filed this. It reproduces on branch `exercise/bughunt-1` (`git switch
exercise/bughunt-1`, `npm start`):

> "If I drag a task's dependency into **Done**, the dependent card keeps showing its
> **blocked** badge. Refreshing the page fixes it. Weirdly, deleting some unrelated task
> also fixes it. No errors in the console."

Rules of engagement — this is a debugging exercise, not a reading exercise:

1. **Do not diff against main.** (`git diff main` finds it in 10 seconds and teaches
   nothing. On a real team there is no "correct branch" to diff against.)
2. Reproduce it first: two tasks, B depends on A, drag A to Done, observe B.
3. Work the method: is the *server* wrong or the *client*? (The Network tab answers
   this in one look — check what `GET /api/tasks` actually returns after the drag.)
   Then: which function decides a badge renders? Breakpoint it. Inspect what it sees.
   Explain the "deleting a task fixes it" clue — that's the fingerprint of the cause.
4. Done means: a one-paragraph root-cause explanation (*why* does it happen, *why* does
   deletion heal it), the fix, and — the senior move — a sentence on how this entire
   bug class gets prevented (hint: what makes a cache key *correct*?).
5. Afterward, and only afterward: `git log` + `git diff main` on the branch to see how
   innocent the offending commit looks. That's the takeaway.

**Research prompts:** `rr` and time-travel debugging; core dumps and how to read one;
flame graphs (Brendan Gregg); OpenTelemetry's traces/spans model; error-tracking
services (Sentry-style) vs logs; why "it works after a refresh" almost always means
stale client state.
