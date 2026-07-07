# iOS Development — The Platform, in Terms You Already Know

iOS is not "a smaller computer" — it's a platform with opinions: about memory, about
your app's lifecycle, about what you're allowed to touch, and about how UI should be
built. This module maps those opinions onto concepts you already have, and hands you a
working project ([TaskFlow iOS](../project-5-taskflow-ios/)) that is a real client of
the TaskFlow API you already built.

---

## 1. The Platform Mental Model

Three constraints shape everything in mobile development:

- **Your app is a guest.** It runs in a sandbox (its own container directory, no
  peeking at other apps), asks permission for anything sensitive (camera, location,
  notifications), and can be *terminated by the OS at any time* to reclaim memory. You
  don't get a say. Design consequence: persist anything you can't afford to lose, and
  treat "app relaunched from nothing" as a normal path, not an edge case.
- **The lifecycle is not main().** Your process transitions between states — active,
  inactive, background, suspended, dead — driven by the user flipping between apps and
  the OS's resource decisions. SwiftUI surfaces this as `scenePhase`; the discipline is
  the same as server statelessness: externalize state you care about.
- **Resources are honest.** Battery, radio, and thermal budgets are real. Polling loops
  and chatty networking that pass unnoticed on a server get your app killed in reviews
  (both the App Store kind and the 1-star kind).

## 2. Swift, for Someone Who Knows Java/C++/Python

Swift will feel like the best parts of all three. What deserves your attention:

- **Value types are the default.** `struct`s are copied on assignment (with
  copy-on-write optimizations), and most of your model layer should be structs. This is
  C++ value semantics with none of the footguns — and it kills a whole class of
  shared-mutable-state bugs. Classes exist for identity and reference semantics; you'll
  use them rarely.
- **Optionals are enforced.** `String?` is `Optional<String>` and the compiler will not
  let you forget the nil case (`if let`, `guard let`, `??`). This is Java's
  `Optional`/`@Nullable` with teeth, and the single biggest crash-class eliminator.
- **Protocols over inheritance.** Swift's `protocol` ≈ interface + extensions +
  associated types. The repository/adapter patterns from this workspace translate
  directly — TaskFlow iOS injects its network transport as a protocol exactly like
  Ledger injects `fetch_json`.
- **ARC, not GC.** Memory is reference-counted at compile-time-inserted points —
  deterministic like C++ `shared_ptr`, no pause-the-world collector. The cost:
  reference *cycles* leak. `weak`/`unowned` break them; retain-cycle hunting in closures
  (`[weak self]`) is the platform's signature memory skill.
- **Structured concurrency** — `async/await` plus **actors** (types whose state is
  protected from data races *by the compiler*). Swift 6 turns data-race safety into
  compile errors, the way Rust made memory safety a compile error. `@MainActor` marks
  UI-touching code — the "UI updates must happen on the main thread" rule, enforced by
  types instead of crash reports.

## 3. SwiftUI — You Already Wrote This By Hand

Remember TaskFlow's `board.js` header comment — *"state → render, events → callbacks →
new state → render... the core idea React formalizes"*? SwiftUI is that idea as a
first-class language feature:

```swift
struct BoardColumn: View {
    let title: String
    let tasks: [Task]              // state in ⤵
    var body: some View {          // UI out — a pure function of its inputs
        ForEach(tasks) { task in
            TaskCard(task: task)   // no .createElement, no .append — you
        }                          // declare WHAT, the framework diffs HOW
    }
}
```

- Views are cheap value-type *descriptions*, recomputed freely; the framework diffs the
  description against the screen and patches minimally — the keyed-diffing upgrade the
  board.js comments said you'd want at 5,000 tasks.
- **State drives everything**: `@State` (view-local), `@Observable` model objects,
  `@Binding` (two-way handles). Mutate state → dependent views recompute. There is no
  `document.getElementById` to drift out of sync; the bughunt-style stale-render bug
  class is *designed away*.
- The catch: you give up imperative control. When SwiftUI recomputes "too much" or
  animations misbehave, debugging means understanding its dependency tracking —
  different skill, same detective method from the debugging module.

## 4. Architecture on iOS — MVVM and Unidirectional Flow

The same layering discipline as everything else in this workspace, with platform names:

```
View (SwiftUI)         declarative, dumb, previews well
   ↕ observes
ViewModel (@Observable, @MainActor)   screen state + intents; no UIKit/SwiftUI imports
   ↕ calls
Domain/API client (TaskFlowKit)       models, business rules, transport — a Swift package
```

TaskFlow iOS splits exactly this way: **TaskFlowKit** is a plain Swift package (builds
and tests headlessly, no simulator needed) holding models, the API client, and the
blocked-task logic; the app target is a thin SwiftUI skin. That boundary is the
repository pattern lesson again — and it's what makes the core testable in CI without a
device farm.

## 5. Networking, Persistence, Testing — the Short Map

- **Networking:** `URLSession` + `async/await`, `Codable` for JSON (compiler-derived
  encode/decode — what you hand-rolled in other projects, generated from your types).
  All the discipline from the [networking module](08-networking-and-apis.md) applies
  unchanged: timeouts, retry classes, adapter boundary. Mobile adds: the radio is
  battery-expensive and *usually absent* — offline isn't an error state, it's Tuesday.
- **Persistence ladder:** `UserDefaults` (small key-value) → files in your sandbox
  container → **SwiftData**/Core Data (object graph + queries) → SQLite directly (your
  Ledger knowledge transfers). TaskFlow iOS's Sprint 2 has the offline-cache story.
- **Testing:** Swift Testing (`@Test`, `#expect`) for units — TaskFlowKit's suite runs
  in seconds; XCUITest drives the real UI in a simulator (slow, top-of-pyramid — same
  economics as E2E everywhere); SwiftUI **Previews** are the fast visual loop.

## 6. Tooling & Distribution Reality

- **Xcode** is non-negotiable: editor, build system, simulators, Instruments (the
  profiler — Time Profiler and Leaks are the two you'll actually use), and the
  signing/provisioning machinery.
- **The `.xcodeproj` problem:** Xcode's project file is a merge-conflict magnet, so
  real teams *generate* it from a declarative spec — this project uses **XcodeGen**
  (`project.yml` is committed; the generated project is gitignored). That decision has
  its own ADR in the repo; the pain it prevents is legendary.
- **Simulator vs device:** the simulator runs your app compiled *for your Mac's CPU* —
  fast, free, and fine for almost everything; a real device is needed for cameras,
  performance truth, and push notifications.
- **Distribution is gated.** Apps are code-signed, reviewed by Apple, and shipped
  through the App Store (TestFlight for betas). Free account: run on your own device;
  $99/year developer program: TestFlight + App Store. Research, don't memorize:
  certificates vs provisioning profiles (everyone relearns these every time).

## 7. Lab Track

| # | Lab | What it teaches |
|---|-----|-----------------|
| 1 | Read TaskFlowKit with its LEARNING_PATH: models → transport protocol → client → blocked logic | The workspace's layering, in Swift |
| 2 | `swift test` the Kit, then run the app in the simulator against your real TaskFlow server (`npm start` — the simulator reaches your Mac's `localhost` directly) | Full-stack loop: your API, your mobile client |
| 3 | Break the contract on purpose: stop the server mid-session, watch the app's error path; compare with what the networking module said clients owe their users | Offline as a first-class state |
| 4 | Sprint 2 stories in the project backlog: pull-to-refresh polish, offline cache, dependency picker | Real feature work in SwiftUI |
| 5 | Instruments: profile the board screen while creating 50 tasks; find the recompute hot spot | Mobile performance discipline |

**Research prompts:** why Apple pushes value semantics (Swift evolution docs); actors
vs locks; SwiftData vs Core Data (2023+ split); what "offline-first" architectures do
(sync engines, CRDTs — deep water, know it exists); Kotlin Multiplatform as the
share-the-core strategy (see the Android module's closing note).
