# Agile & Working on a Team

How software teams actually organize work — and how to simulate it while working solo.

---

## 1. Where Agile Came From (the 60-second version)

The classic model — **Waterfall** — runs a project as one long pipeline: gather *all*
requirements → design *everything* → build → test → ship. It fails for most software
because requirements are wrong or stale by the time you ship, and you learn the most
important things *by building*, which Waterfall postpones feedback on until the end.

The **Agile Manifesto** (2001) is a reaction to that. Its core bet: **short feedback
loops beat long-range planning.** Ship something small, learn from it, adjust. Everything
else — sprints, standups, story points — is machinery in service of that one bet.

Waterfall isn't dead, and isn't stupid: firmware in a pacemaker, or anything with a
regulator or a fixed contract, still front-loads specification. Know both; pick by context.

## 2. Scrum: the vocabulary you'll hear everywhere

Scrum is the most common Agile framework. Even teams that don't "do Scrum" use its words.

**Roles**

- **Product Owner (PO)** — owns *what* to build and in what order. Keeps the backlog
  prioritized. Says "users need X because Y"; never says "use Postgres."
- **Scrum Master** — owns the *process*: runs ceremonies, removes blockers. Often a
  part-time hat, not a full-time person.
- **Developers** — own *how*. Estimate the work, design it, build it, test it.

**Artifacts**

- **Product backlog** — the ordered list of everything the product might need. Alive,
  constantly re-prioritized.
- **Sprint backlog** — the slice the team committed to for this sprint.
- **Increment** — the working software at the end of the sprint. "Working" is the key
  word: half-built features that don't run don't count.

**Ceremonies** (for a 2-week sprint, the most common length)

| Ceremony | When | Purpose | Solo version |
|---|---|---|---|
| Sprint planning | Day 1, ~2h | Pick stories from the backlog, break them into tasks | Read the backlog, pick 1–3 stories, write down your plan |
| Daily standup | Daily, 15 min | Each person: what I did / what I'll do / what's blocking me | Two sentences in a `standup.md` journal — seriously, do this; it exposes stalls |
| Sprint review / demo | Last day | Show working software to stakeholders | Demo to a friend, or record a 2-min screen capture explaining what you built |
| Retrospective | Last day | What went well / what didn't / what we'll change | Answer those three questions in writing; carry ONE change into the next sprint |

## 3. User Stories

The standard unit of backlog work. The format forces you to name the user and the value:

> **As a** [kind of user], **I want** [capability], **so that** [benefit].

Example from the TaskFlow backlog:

> As a team member, I want tasks to show which tasks block them, so that I don't start
> work that can't be finished.

A story is *not* a spec. The spec lives in its **acceptance criteria** — testable
statements that define "done for this story":

- Given a task with an incomplete dependency, the UI shows a "blocked" badge.
- The API rejects a dependency that would create a cycle, with a 422 and a clear message.

Good stories follow **INVEST**: **I**ndependent, **N**egotiable, **V**aluable,
**E**stimable, **S**mall, **T**estable. When a story fails INVEST — usually too big —
split it. Every SPRINT_BACKLOG.md in this workspace uses this exact structure, so you'll
get plenty of reps reading them.

**Epics** are just stories too big to fit in a sprint; they get split into stories.

## 4. Estimation & Story Points

Teams estimate stories in **points** (usually the Fibonacci-ish scale 1, 2, 3, 5, 8, 13),
not hours. Points measure *relative* size + complexity + uncertainty. Why not hours?
Because humans are terrible at absolute estimates but decent at "this is roughly twice
that." A team's **velocity** (points completed per sprint) emerges after a few sprints
and makes forecasting honest.

**Planning poker:** everyone reveals an estimate simultaneously (prevents anchoring),
then the high and low estimators explain their reasoning. The disagreement is the
valuable part — it surfaces hidden assumptions.

Solo: estimate every story before starting, record actuals, and review the gap at your
retro. You will be wrong in a consistent direction. Knowing your direction is the skill.

## 5. Definition of Done

A team-wide checklist that applies to *every* story, on top of its acceptance criteria.
A typical one (and the one used in these projects' CONTRIBUTING.md files):

- [ ] Code merged to `main` via a reviewed pull request
- [ ] New/changed behavior covered by tests; all tests green
- [ ] No linter/compiler warnings introduced
- [ ] Docs updated (README, ARCHITECTURE, or ADR if a decision changed)
- [ ] The feature was actually run by a human, not just tested

"Done" ≠ "code written." The gap between those two is where production incidents live.

## 6. Kanban: the other major flavor

Kanban drops sprints entirely: work flows continuously across a board
(`To Do → In Progress → Done`), with **WIP limits** (e.g., max 2 items in progress per
person). Finish things before starting things. It suits support/ops teams and solo work
very well — TaskFlow (project 2) is literally a Kanban board app, chosen so you build
the tool while learning the method.

Scrum vs Kanban isn't a religion. Many teams run "Scrumban." The underlying principles
are the same: small batches, visible work, finish before you start, reflect and adjust.

## 7. The Rituals Around the Code

Stuff that happens on real teams that no course mentions:

- **Backlog grooming/refinement** — a mid-sprint meeting to clarify and estimate
  *upcoming* stories so sprint planning isn't chaos.
- **Tech debt negotiation** — engineers lobby the PO for capacity to fix internal
  quality issues. Good teams budget ~20% for it. The projects' backlogs include tech-debt
  stories so you practice recognizing them.
- **Blameless post-mortems** — after an incident: what happened, why (systemically, not
  who), what changes. The habit of writing things down after failure is a superpower.
- **On-call** — many teams rotate responsibility for production alerts. Design
  consequence: engineers who get paged write more defensive code.

## 8. Your Solo Sprint Loop

The concrete cadence to use for the projects here (1-week sprints suit solo pace):

1. **Plan** (30 min): pick 1–3 stories from the project's SPRINT_BACKLOG.md. Write your
   task breakdown.
2. **Work** in feature branches. Two-line standup journal each session.
3. **Review**: before merging, read your own full diff (`git diff main...`) against the
   CONTRIBUTING.md checklist — or have an AI review it and argue back.
4. **Demo**: run the app end to end; capture what you'd show a stakeholder.
5. **Retro** (15 min, written): went well / didn't / one change for next sprint.

It will feel like ceremony overhead for a party of one. That's partly the point — you're
building the muscle memory for the team versions, and you'll be surprised how much the
written standups and retros improve your own throughput.
