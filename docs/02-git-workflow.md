# Git Workflow — How Teams Actually Use Version Control

You know `git add/commit/push`. This doc covers the layer above: the conventions and
workflows that let 2–200 people share a repo without chaos.

---

## 1. The Mental Model (worth 10 minutes even if you know git)

- A **commit** is a snapshot of the whole tree plus a pointer to its parent(s). The
  history is a DAG — the same structure you studied in algorithms.
- A **branch** is just a movable label pointing at a commit. Cheap. Disposable. Teams
  create dozens per week.
- **`merge`** joins two histories with a merge commit. **`rebase`** replays your commits
  on top of another branch, rewriting them (new hashes). Rule of thumb every team
  agrees on: *never rewrite history that's been pushed and shared* — rebase your own
  unpushed work freely, never `main`.

## 2. Branching Strategies

The big three. Every team picks one (or drifts into one accidentally):

### GitHub Flow — the default choice, and what these projects use
- `main` is always deployable/green.
- Every change: branch off `main` → commits → pull request → review → merge → delete branch.
- Simple, low ceremony, pairs naturally with continuous deployment.

### Trunk-Based Development
- Everyone commits to `main` (the "trunk") directly or via branches that live *hours*, not days.
- Unfinished features hide behind **feature flags** (runtime switches) rather than long branches.
- Requires excellent automated tests and CI. It's what elite delivery teams converge on
  (see the DORA research — worth a search: "DORA metrics").

### GitFlow
- Heavyweight: permanent `main` + `develop` branches, plus `feature/*`, `release/*`, `hotfix/*`.
- Designed for versioned, boxed releases (e.g., a desktop app shipping v2.3). Overkill
  for continuously deployed services, and its own author now says so — but you'll meet
  it in older organizations, so recognize it.

**Why long-lived branches hurt:** two branches that diverge for weeks develop conflicting
assumptions, and the eventual merge is a bug factory. Every strategy above is a different
answer to "how do we keep integration pain small and frequent instead of huge and rare."

## 3. What Makes a Good Commit

- **Atomic**: one logical change. If your message needs "and," split it.
- **Complete**: the code compiles and tests pass at every commit. (Bisecting a broken
  history to find a bug — `git bisect` — only works if each commit builds.)
- **Tests ship with the feature** in the same commit/PR — not "later."

### Conventional Commits (used throughout these projects)

A machine-readable message format — `type(scope): summary`:

```
feat(api): add suggested-order endpoint using topological sort
fix(board): prevent drag-and-drop onto the same column
docs(adr): record decision to use JSON file storage
refactor(service): extract validation from route handlers
test(repo): cover cycle detection edge cases
chore: bump express to 4.19
```

Why bother: changelogs can be generated automatically, semantic-version bumps can be
inferred (`feat` → minor, `fix` → patch, `feat!`/`BREAKING CHANGE` → major), and history
becomes scannable. Read each project's history with `git log --oneline` and you'll see
the story of its construction.

The body (separated by a blank line) explains **why**, not what — the diff already shows
what. Reference the story: "Implements S2-1."

## 4. Pull Requests & Code Review

The PR is where most engineering communication happens. Norms that make it work:

**As author**
- Keep PRs small: ~200–400 changed lines reviews well; 2,000 lines gets rubber-stamped.
- Write a real description: what, why, how to test it, screenshots if UI.
- **Review your own diff first.** You'll catch half the issues before anyone else looks.
- Respond to feedback with either a change or a reason — never silence.

**As reviewer**
- Review promptly (a blocked teammate is expensive) and kindly ("what do you think about
  X?" beats "this is wrong").
- Look for: correctness, missing tests, unclear naming, hidden coupling, security holes.
  Do NOT nitpick formatting — that's the linter's job, not a human's.
- Approve when it's *better than main and safe*, not when it's perfect.

Each project's CONTRIBUTING.md has a concrete review checklist. Practice on your own
diffs: open one, read it cold the next morning, and be your own harshest reviewer.

## 5. Releases, Tags, and Semantic Versioning

- **SemVer**: `MAJOR.MINOR.PATCH` — breaking / new-but-compatible / bug-fix. Version
  numbers are *communication to consumers*, not marketing. Pre-1.0 means "no stability
  promises."
- **Tags** mark releases: `git tag -a v0.1.0 -m "..."`. Each project here is tagged
  `v0.1.0` at the end of Sprint 1 — `git tag` then `git show v0.1.0` to inspect.
- A **CHANGELOG** tells users what changed. With conventional commits it can be generated.

## 6. The Setup in These Projects

Each project repo follows the same conventions — this is your "team's git policy":

- **Strategy:** GitHub Flow. `main` is always green.
- **Commits:** Conventional Commits, tests included with features.
- **History as curriculum:** `git log --oneline --graph --all` in each repo shows the
  scaffold being built in reviewable increments. Read the history before the code — it
  answers "where do I even start?" better than any tutorial.
- **Open feature branch:** each repo has a `feature/s2-*` branch already started with a
  stub and TODOs — your on-ramp for Sprint 2. Check it out, finish it, merge it to
  `main` as if a reviewer approved it, then delete the branch.
- **Tags:** `v0.1.0` marks the Sprint 1 increment.

## 7. Daily-Driver Commands Beyond the Basics

```bash
git log --oneline --graph --all      # see the whole DAG — use constantly
git diff main...HEAD                 # everything your branch changes (the "PR diff")
git add -p                           # stage hunk-by-hunk → forces you to review yourself
git stash / git stash pop            # shelve WIP to switch context
git bisect start                     # binary-search history for the commit that broke it
git blame <file>                     # who last touched each line (find the WHY via its commit)
git restore --staged <f> / git restore <f>   # unstage / discard changes safely
git rebase -i HEAD~3                 # squash/reword your own unpushed commits before PR
git reflog                           # the undo log — how you recover from "I destroyed it"
```

**Research prompts:** `git bisect` walkthrough; merge vs rebase vs squash-merge PR
strategies (each project uses merge commits — form an opinion); what `git rerere` does;
how feature flags enable trunk-based development.
