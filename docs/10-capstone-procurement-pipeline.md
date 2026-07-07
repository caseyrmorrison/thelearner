# Capstone — The Procurement Pipeline (Stockroom ⇄ Ledger)

Everything so far has been a single program talking to itself. Real systems are
*federations*: your code calling services owned by other teams, across a network that
fails, against contracts that drift. This capstone makes two of your projects into such
a federation — and you own both sides, so you get to feel the producer's and consumer's
pain in the same week.

**The scenario:** Sam (Ledger's persona) buys workshop supplies from the warehouse that
Stockroom manages. Stockroom exposes its first real API; Ledger grows a `purchase`
command that orders through that API and records the expense. One transaction of
business value, two processes, one HTTP contract between them.

```
┌──────────────────────┐         HTTP (localhost)          ┌──────────────────────┐
│  Ledger (Python CLI) │  POST /api/v1/orders              │ Stockroom (Java)     │
│                      │ ────────────────────────────────► │                      │
│  cli → service ──►   │          201 {order, total}       │  api layer (NEW)     │
│  stockroom adapter   │ ◄──────────────────────────────── │   └─► OrderService   │
│   └─► record txn     │                                   │        (existing!)   │
└──────────────────────┘                                   └──────────────────────┘
```

The prime directive on both sides: **the existing service layers don't change.** The
API layer is a new *presentation* skin over Stockroom's services; the adapter is a new
*infrastructure* seam under Ledger's. If either team's domain code has to know this
integration exists, the layering has failed. (This is the payoff moment for every
ARCHITECTURE.md you've read.)

Prerequisites: both projects' Sprint 2 experience, plus handbook modules on
[architecture](04-architecture-guide.md) (§4), [networking](08-networking-and-apis.md)
(all of it — this capstone is its final exam), and [debugging](09-debugging-and-observability.md)
(§5, you'll want request logging when the drills start).

---

## C-1 — Author the contract, before any code (3 points, both teams)

> As both teams, we want a written API contract agreed before implementation, so that
> neither side discovers the other's assumptions in an integration failure.

The contract is the real interface; the code on both sides is just an implementation of
it. In industry this is "contract-first" (vs "code-first, docs maybe"), and the fights
it front-loads are exactly the fights you want early.

**Deliverable:** `docs/contracts/stockroom-api-v1.md` in this handbook repo — neutral
ground, because a contract between two teams belongs to *neither team's* repo (in a
real org: an API registry, an OpenAPI file in a shared repo, or a protobuf schema repo).

**Acceptance criteria**
1. Specifies `GET /api/v1/products` (list: sku, name, unit price, available stock) and
   `POST /api/v1/orders` (request: order lines + client-supplied **idempotency key**;
   response: allocations, shortfalls, total).
2. Every response documented with a real JSON example, including *every* error:
   400 (malformed), 404 (unknown SKU), 409 (idempotency-key replay with different
   body), 422 (valid shape, unfulfillable — mirrors Stockroom's `Rejected` result).
3. Money crosses the wire as `{"amount": "63.92", "currency": "USD"}` — string, never a
   JSON number. Both teams' handbooks agree on why (Ledger ADR-0002; JSON numbers are
   floats). Partial fulfillment representation is decided *here*, not improvised later.
4. The idempotency-key semantics are written down precisely: same key + same body =
   replay the stored response (200, not a second order); same key + different body =
   409. This sentence will save you in C-4.
5. Version `v1` is in the path, and the contract states the compatibility promise:
   additive changes allowed, breaking changes require `v2` (C-5 will test your resolve).

**Research:** OpenAPI/Swagger (write this contract as prose now; translating it to
OpenAPI yaml is a fine bonus exercise); Stripe's idempotency-key docs (the industry's
reference implementation of the idea).

## C-2 — Stockroom serves the API (5 points, Java side)

> As an API consumer, I want Stockroom's catalog and ordering exposed over HTTP per the
> contract, so that programs — not just the console menu — can transact with the warehouse.

**Framework decision, made for you (and worth an ADR in Stockroom):** use the JDK's
built-in `com.sun.net.httpserver.HttpServer`. No Spring, no dependencies. It's ~40
lines to stand up, keeps the focus on *the contract and the layering* instead of
framework magic, and makes Spring Boot (already in your icebox as a research story)
mean something when you later see what it automates away.

**Acceptance criteria**
1. `java -jar ... --serve` (alongside the existing `--demo`) starts the server; port
   from `STOCKROOM_PORT` env var, default 8484 (12-factor, as ever).
2. A new `api/` package is pure presentation: parse/validate HTTP → call the *existing*
   `InventoryService`/`OrderService` → translate results (including the sealed
   `OrderResult` — an exhaustive `switch` mapping `Fulfilled`/`PartiallyFulfilled`/
   `Rejected` to contract responses/status codes; the compiler will not let you forget
   a case, which is the whole reason it's sealed).
3. Wire-format compliance: responses match C-1's documented examples byte-for-shape,
   money as strings. JSON is hand-rolled at this scale — with a comment acknowledging
   that real teams reach for Jackson the moment strings need *escaping*, and a test
   proving a product named `Say "cheese"` doesn't produce invalid JSON (that test will
   fail your first implementation; that's the point).
4. Idempotency keys honored per contract: a replay store (in-memory `Map` is fine)
   returns the original response for a repeated key; conflicting body → 409.
5. Integration tests boot the real `HttpServer` on port 0 and exercise every contract
   example — happy paths *and* all four error codes. `./mvnw test` stays green.
6. README gains a "Serve mode" section with a verified curl session.

**Research:** `com.sun.net.httpserver` javadoc (`HttpServer.create`, `HttpContext`,
`HttpExchange`); why the JDK shipped an HTTP *server* nobody talks about; virtual
threads (`Executors.newVirtualThreadPerTaskExecutor()` as the server executor — a
one-line taste of modern Java concurrency).

## C-3 — Ledger purchases through the API (5 points, Python side)

> As Sam, I want `ledger purchase --sku SKU-1002 --qty 3` to place the order with the
> warehouse and record what I actually paid, so that buying and bookkeeping are one
> action instead of two chances to forget.

**Acceptance criteria**
1. A new adapter (`src/ledger/stockroom.py`) is the only file that knows the API exists
   — same shape as S3-1's rates adapter (if you built that first, this will feel like a
   rerun; that's the pattern working). Base URL from `--api-url` / `STOCKROOM_API_URL`,
   default `http://localhost:8484`.
2. Full client discipline from the networking module: explicit timeouts; retries with
   backoff+jitter **only** on the retryable class; and — because ordering is a POST —
   every retry carries the **same idempotency key** (generated once per command
   invocation, `uuid4`). A test proves the retry reuses the key.
3. Outcomes map to honest UX: `Fulfilled` → transaction recorded (category `supplies`,
   description naming the order), print confirmation; `PartiallyFulfilled` → record
   only the *shipped* total, warn on stderr about shortfalls; `Rejected`/422 → record
   nothing, exit 1 with the reason; network-dead → record nothing, exit 1, clear
   message. The invariant a test pins: **no transaction is ever recorded for money the
   warehouse didn't charge.**
4. Tests use an injected fake transport (no real network in `pytest`); cover: fulfilled,
   partial, rejected, timeout-then-retry-same-key, 409 replay conflict, malformed JSON.
5. One manually-run end-to-end script (`scripts/e2e-purchase.sh` or a README session):
   start Stockroom `--serve`, run a real `ledger purchase`, show the transaction in
   `ledger list`. This is the demo you'd give at sprint review.

**The uncomfortable question (answer it in a comment or short ADR):** what if the order
succeeds and *then* the local ledger write fails? You've just met the **dual-write
problem** — the reason distributed systems people lose sleep. At this scale a clear
error message telling Sam to record manually is a legitimate answer; *knowing you made
that trade-off* is the learning objective. Research: "dual write problem", sagas,
transactional outbox — recognize the vocabulary, don't build it.

## C-4 — Failure drills (3 points, both teams)

> As both teams, we want to have *watched* the integration fail in every predicted way,
> so that the error handling is tested by reality instead of by optimism.

Chaos engineering at teaching scale. Deliverable: a lab log
(`docs/labs/failure-drills.md` in either repo) recording each drill — prediction,
observation, and any fix that fell out.

**The drills**
1. Purchase with Stockroom **not running** → clean Ledger error, nothing recorded.
2. Start Stockroom, `kill -9` it mid-purchase (a `sleep` in the handler makes the window
   generous) → does the retry logic behave? What does Sam see?
3. Retry storm: same idempotency key POSTed 5× → exactly one order exists (check
   Stockroom's state); different key each time → five orders, which is *correct* and
   is why the key must survive retries (compare with your C-3 test's assumptions).
4. Slow server (2s `sleep` in the handler) vs your client timeout → tune the timeout
   deliberately and write down the number and the reasoning.
5. Malformed response (temporarily make Stockroom return truncated JSON) → Ledger must
   fail with a clear message, not a stack trace.

**Acceptance criteria:** the log exists with all five drills; at least one drill found a
real defect (if all five pass untouched on the first run, be suspicious — drill 2 or 5
almost always draws blood); every defect found got a regression test.

## C-5 — Survive a breaking change (3 points, both teams)

> As the API's owners, we want to rename `unitPrice` to `unit_price` (the consumer team
> "won" a naming-convention argument), so that we learn what API evolution actually costs.

Do it wrong, then right:

1. **Wrong first (on a branch):** rename the field in Stockroom's v1 responses, deploy,
   run `ledger purchase`. Watch a consumer you didn't warn break at runtime. Note *how
   far from the cause* the client's error appears — that distance is why API breakage
   is so expensive to diagnose in the wild.
2. **Right:** per the C-1 compatibility promise, either (a) additive — serve both fields
   in v1, deprecate the old one in the contract, migrate the client, remove the old
   field in a documented `v2` — or (b) stand up `/api/v2/` beside v1. Choose, implement,
   and record the choice as an ADR (in the handbook, next to the contract — decisions
   about shared contracts live on neutral ground too).

**Acceptance criteria:** the "wrong" branch exists with the observed failure pasted into
its final commit message; main lands the "right" path with contract + client + ADR in
sync; both repos' tests green throughout the "right" path (that's what "non-breaking"
means, mechanically).

---

## Sequencing & scale

C-1 → C-2 → C-3 → C-4 → C-5, one story per solo sprint. ~19 points total: roughly a
month of evening-paced work, and the closest thing this workspace has to real industry
experience. Both project backlogs reference this document; the day you finish C-5,
write the retro — then compare it to what the handbook's agile doc promised retros
would do for you back when this was all theory.
