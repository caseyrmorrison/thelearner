# Android Development — The Platform, in Terms You Already Know

Android and iOS solve the same problems with different personalities: Android is
JVM-family (your Java knowledge compounds), more open (sideloading, multiple stores,
deeper OS hooks), and more fragmented (thousands of device models, API levels spanning
a decade). This module maps the platform, and pairs with a working project
([TaskFlow Android](../project-6-taskflow-android/)) — a Jetpack Compose client of the
same TaskFlow API your iOS app talks to. Read the [iOS module](11-ios-development.md)
too, even if you only care about Android: the comparison is where the understanding is.

---

## 1. The Platform Mental Model

Same three constraints as iOS — sandboxed guest, OS-driven lifecycle, honest resource
budgets — plus Android's own flavor:

- **Components, not just an app.** An Android app is a set of components the OS
  instantiates: Activities (screens), Services (background work), BroadcastReceivers
  (system events), ContentProviders (data sharing). The OS can start your app *at any
  of these entry points* — there is no single `main()` you control.
- **Process death is routine.** Rotate the phone and your Activity is destroyed and
  recreated; background the app and the process may be killed, then resurrected with
  the user expecting their half-typed form intact. `ViewModel` (survives rotation) and
  `SavedStateHandle` (survives process death) exist precisely for this — Android's
  hardest-won lesson, now mostly solved if you use the architecture components.
- **Fragmentation is the tax.** `minSdk` vs `targetSdk`, vendor-skinned OS versions,
  screen sizes from watch to fold-out tablet. The ecosystem's answer: Jetpack libraries
  that backport behavior, and designing responsively from day one.

## 2. Kotlin, for Someone Who Knows Java (and Just Wrote Modern Java)

Kotlin is what Stockroom's "modern Java" features are converging toward, arrived a
decade early. Direct translations from what you just used:

| You used in Stockroom | Kotlin equivalent |
|---|---|
| `record Money(...)` | `data class Money(...)` — same value semantics, plus `copy()` |
| `sealed interface OrderResult permits...` + exhaustive `switch` | `sealed interface` + exhaustive `when` — same make-illegal-states-unrepresentable move |
| `Optional<Product>` returns | Nullable types `Product?` with compiler-enforced handling (`?.`, `?:`, `let`) — nullability in the type system, no wrapper |
| Streams (`filter/map/collect`) | Collection functions (`filter/map/groupBy`) — no `.stream()`/`.collect()` ceremony |
| `PricingStrategy` interface | Function types `(Order) -> Money` or interfaces — functions are first-class |

New muscle, no Java analogue:

- **Coroutines** — Kotlin's structured concurrency. `suspend fun` ≈ Swift's `async`;
  a `CoroutineScope` owns the lifetime of work launched in it, so a closed screen
  cancels its own in-flight requests (`viewModelScope` does this automatically — the
  fix for a classic leak class). `Dispatchers.Main` vs `IO` is the thread-hopping
  vocabulary.
- **Flow** — cold async streams; `StateFlow` is the "observable current value" that
  ViewModels expose and Compose collects. It's the state→render pipe, typed.

## 3. Jetpack Compose — Same Revolution, Kotlin Accent

Compose is to Android what SwiftUI is to iOS, and both are what `board.js` hand-rolled:
UI as a function of state.

```kotlin
@Composable
fun BoardColumn(title: String, tasks: List<Task>, onTaskClick: (Task) -> Unit) {
    Column {
        Text(title, style = MaterialTheme.typography.titleMedium)
        LazyColumn {                      // lazy = virtualized, like a RecyclerView
            items(tasks, key = { it.id }) { task ->
                TaskCard(task, onClick = { onTaskClick(task) })
            }
        }
    }
}
```

- `@Composable` functions *emit* UI; changing observed state triggers **recomposition**
  of exactly the affected functions. The `key = { it.id }` is the keyed diffing the
  board.js comments promised you'd meet again.
- **State hoisting** is the local idiom: composables receive state and emit events
  upward (`onTaskClick`) — the same one-way flow as board.js's callbacks, formalized.
  A composable that owns its own hidden state is the code smell.
- Compose replaced a 15-year XML-layout system (`findViewById`, inflation, fragments
  full of lifecycle traps). You'll still see XML views in older codebases — recognize,
  don't learn first.

## 4. Architecture on Android — the Same Diagram Again

```
Composables (UI)           dumb, previewable, state in / events out
   ↕ collects StateFlow
ViewModel                  screen state + intents; survives rotation; viewModelScope
   ↕ suspend calls
:core module (pure Kotlin) models, API client, blocked-task logic — no Android imports
```

TaskFlow Android enforces the bottom boundary with the build system itself: **`:core`
is a plain Kotlin/JVM Gradle module** — it *cannot* import Android classes, compiles
without the Android SDK, and its tests run on your desktop JVM in seconds. The `:app`
module holds Compose + ViewModels. Same discipline as TaskFlowKit on iOS, same reason
the repository pattern kept SQL out of Ledger's domain: the build fails if the layering
lies.

## 5. Networking, Persistence, Testing — the Short Map

- **Networking:** the project uses `HttpURLConnection`-free modern stack — an injected
  transport interface with a real implementation (OkHttp or `java.net.http`) so tests
  fake the seam, per the [networking module](08-networking-and-apis.md). JSON via
  `kotlinx.serialization` (`@Serializable data class` — compiler-derived like Codable).
  **The emulator gotcha everyone hits once:** inside the Android emulator, `localhost`
  is the *emulator itself*; your Mac's localhost is **`10.0.2.2`**. The README makes
  this configurable; now you know why.
- **Persistence ladder:** `DataStore` (small typed key-value) → **Room** (SQLite with
  compile-time-checked SQL — your Ledger schema instincts apply directly) → SQLite raw.
  Sprint 2's offline-cache story uses Room.
- **Testing:** JUnit on the JVM for `:core` (fast, no device); Robolectric to run
  Android-dependent code on the JVM; Compose UI tests and Espresso on emulator/device
  (top of the pyramid, same slow-but-honest economics as ever).

## 6. Tooling & Distribution Reality

- **Gradle is the build system** — same wrapper pattern as Stockroom's `mvnw`
  (`./gradlew` is committed; first run downloads everything). Android Gradle Plugin
  versions pair tightly with Gradle versions; the project pins known-good pairs and the
  ADR explains the compatibility dance. Command-line builds work without opening
  Android Studio: `./gradlew :core:test` (pure JVM) and `./gradlew :app:assembleDebug`
  (produces an installable APK).
- **`local.properties`** points at your SDK (`sdk.dir=...`) and is gitignored on every
  Android team on Earth — machine-specific config, the same reasoning as `.env`.
- **Android Studio** = IntelliJ + Android tooling: layout preview, emulator (AVD
  Manager), profiler, APK inspector. The debugger is the IntelliJ one the debugging
  module already recommended for Stockroom.
- **Distribution:** sign with your keystore (lose it historically = lose your app
  identity; modern Play App Signing mitigates), upload an AAB to the Play Console,
  review is faster/looser than Apple's, staged rollouts are built in. Sideloading and
  F-Droid exist — Android's openness is real, with malware as its price.

## 7. Lab Track

| # | Lab | What it teaches |
|---|-----|-----------------|
| 1 | Read `:core` with the LEARNING_PATH, then diff it mentally against TaskFlowKit — same design, two languages | The architecture is the constant; syntax is the variable |
| 2 | `./gradlew :core:test`, then run the app on an emulator against your real server (base URL `http://10.0.2.2:3000`) | Full-stack loop + the emulator networking lesson |
| 3 | Rotate the device mid-session; background the app for an hour; kill the process from the overview screen and relaunch | Lifecycle survival — what ViewModel does and doesn't save |
| 4 | Sprint 2 stories in the project backlog: offline cache with Room, swipe-to-change-status, dependency picker | Real feature work in Compose |
| 5 | Android Studio profiler while scrolling a 200-task board; find the recomposition storm (Layout Inspector shows recomposition counts) | Compose performance literacy |

**Research prompts:** how Compose recomposition actually decides what to re-run
(stability, `remember`); WorkManager for deferrable background jobs; why Google moved
from Java to Kotlin officially (2017–2019 history); **Kotlin Multiplatform** — sharing
`:core`-style modules across Android *and* iOS, which is exactly the boundary both
mobile projects drew... on purpose. Compare and form an opinion: is KMP's promise the
repository pattern at ecosystem scale, or a coupling trap?
