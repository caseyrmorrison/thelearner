# Networking & API Integration — Talking to Other Programs

Half of backend engineering is being a *server*; the other half is being a *client* of
someone else's server — a payments API, an auth provider, a rates feed. TaskFlow already
teaches the server side. This module covers the whole conversation, with special weight
on the client side, because that's where school coverage is thinnest and where
production incidents are born.

---

## 1. The Stack, From an Engineer's Chair

You've had the networking course; here's the slice you touch daily. Your JSON rides in
an HTTP message, inside a TLS session, over a TCP connection, addressed by IP, found via
DNS. What that layering buys you day-to-day:

- **DNS** is why "the API is down" is sometimes "DNS is misconfigured" (`dig api.example.com`).
- **TCP** gives ordered, reliable bytes — and connection setup costs a round trip, which
  is why clients reuse connections (keep-alive, pooling).
- **TLS** gives privacy/integrity — and *another* round trip, and certificate expiry as
  a classic outage cause. Never disable certificate verification to "fix" an error.
- **HTTP** is the application vocabulary — the layer you actually design in.

Research when curious, not before: the TLS handshake, HTTP/2 multiplexing, what a socket
actually is (file descriptor + kernel buffers).

## 2. HTTP Anatomy — Read One Message Carefully

```
POST /api/tasks HTTP/1.1            GET is "read, safe to repeat"
Host: localhost:3000                the one mandatory header
Content-Type: application/json     "here's how to parse my body"
Authorization: Bearer eyJhbGc...    credentials ride in a header
Content-Length: 52

{"title":"Design schema","priority":"high"}
---
HTTP/1.1 201 Created                status: machine-readable outcome
Location: /api/tasks/7              where the new resource lives
Content-Type: application/json

{"id":"7","title":"Design schema",...}
```

Headers worth knowing on sight: `Content-Type`/`Accept` (content negotiation),
`Authorization`, `Cache-Control`/`ETag` (caching, §5), `Location` (redirects & created
resources), `Retry-After` (rate limiting, §4). **Lab 1 makes this concrete against your
own TaskFlow server.**

For *designing* APIs — resources, verbs, status-code vocabulary, statelessness — see the
[architecture guide §4](04-architecture-guide.md) and TaskFlow's README/code; this doc
won't repeat it. Two additions once you're consuming as well as serving: **pagination**
(no real API returns everything; look at `Link` headers and cursor params) and
**versioning** (breaking a published contract breaks strangers' code — hence `/v1/`).

## 3. Consuming APIs — the Discipline

The core mindset shift: **the network is not a function call.** It fails partially, it
fails slowly, and it fails at 2 a.m. Every rule below exists because someone's pager
went off.

### Timeouts — the non-negotiable one
Every network call gets an explicit timeout. Most HTTP libraries default to *infinite*
(Python `requests`/`urllib`: yes, really), so a hung upstream turns your app into a
zombie: threads pile up waiting forever, and *their* callers hang too — failure
propagates upstream like a stuck pipeline. A timeout converts "mystery hang" into a
clean, handleable error. Pick values deliberately: connect timeout short (~3s), read
timeout based on the endpoint's honest latency.

### Retries — helpful, then catastrophic
Transient failures (dropped connection, 503) deserve a retry. But naive retry loops
*amplify* outages: a struggling server receives 3× traffic from everyone retrying — the
thundering herd. Discipline:

- Retry only what's safe to repeat: idempotent requests (GET, PUT, DELETE). Retrying a
  POST can double-charge a card — that's why payment APIs accept **idempotency keys**
  (research Stripe's).
- **Exponential backoff with jitter**: wait 1s, 2s, 4s… ±random. The jitter matters —
  without it, every client retries in synchronized waves.
- Cap attempts (3 is a sane default) and give up loudly.
- Classify before retrying: 4xx means *your request is wrong* — retrying identical
  garbage yields identical garbage (except **429**); 5xx and network errors are the
  retryable class.

### Rate limits
APIs defend themselves with quotas. A **429 Too Many Requests** (often with
`Retry-After: n`) is an instruction, not an insult — honor it. Beyond that: circuit
breakers (stop calling a dying service entirely for a cooldown) are the next concept
out; research when you first integrate something flaky.

### The adapter boundary (architecture meets networking)
Exactly one module in your codebase should know a given external API exists — its URLs,
its auth, its JSON shapes. It translates external DTOs into *your* domain types at the
boundary (the "anti-corruption layer"; it's the ports-and-adapters idea from the
architecture guide again). Payoffs: the vendor's breaking change is a one-file fix,
tests fake one seam, and vendor lock-in gets a price tag. Ledger's Lab 3 builds exactly
this shape.

## 4. Authentication — Proving Who's Calling

- **API keys**: a static secret in a header. Simple, fine machine-to-machine. Keys live
  in env vars / secret managers — never in code, never in git (see practices doc §7).
- **Bearer tokens / JWT**: short-lived signed tokens; the server verifies the signature
  instead of hitting a session store — that's the "stateless" trade (and why revocation
  is JWT's hard problem).
- **OAuth 2.0** in one paragraph: the answer to "let app A act on my app-B account
  without giving A my password." You've used it with every "Sign in with Google" button.
  The dance: A redirects you to B → you consent at B → B redirects back with a code → A
  exchanges the code for a scoped access token. Implement it once (icebox) and the fog
  clears; until then, recognize the vocabulary: client id/secret, scopes, refresh tokens.

## 5. Caching, Push, and Formats — the Rest of the Toolkit

- **HTTP caching:** responses carry `Cache-Control` (how long am I fresh?) and `ETag`
  (a content fingerprint). A client re-asks with `If-None-Match: <etag>`; the server's
  `304 Not Modified` saves the body transfer. Also: cache expensive upstream answers
  locally with a TTL — Ledger's Lab 3 does this for exchange rates.
- **Polling vs webhooks vs sockets:** polling ("anything new?" every n seconds) is
  simple and wasteful; **webhooks** invert it — you expose an endpoint, they POST events
  to you (plus signature verification, because now strangers can hit that endpoint);
  **WebSockets/SSE** hold a connection open for real-time push (TaskFlow's live-updates
  icebox story).
- **Formats:** JSON is the default. Its trap — numbers are IEEE floats — is why Ledger's
  S2-3 serializes money as strings and Stripe sends integer minor units. Binary formats
  (protobuf, MessagePack) buy size/speed/schemas at the cost of human-readability; you'll
  meet protobuf with gRPC in service-to-service ecosystems.

## 6. Testing Code That Talks to the Network

Unit tests must not hit real APIs (slow, flaky, rate-limited, needs secrets in CI). The
seam is the adapter from §3:

- Inject the HTTP-transport function/client into your adapter (same DI move as
  repositories); tests inject a fake returning canned responses — including the ugly
  ones: 429, 500, timeout, malformed JSON. **Test the failure paths; the happy path is
  the easy 20%.**
- Record/replay tooling (VCR-style cassettes) exists for higher-fidelity tests; contract
  tests verify both sides of an API agree on shapes. Research after Lab 3.
- Keep one *manually run* smoke test that hits the real API, excluded from CI.

## 7. Tooling You Should Be Fluent In

```bash
curl -v https://api.frankfurter.dev/v1/latest?base=USD   # -v shows the full conversation
curl -i ...          # response headers with body
curl -X POST -H 'Content-Type: application/json' -d '{...}' ...
curl ... | jq '.rates.EUR'                                # JSON surgery on any output
time curl -m 5 ...   # -m = max seconds: timeouts exist in curl too
```

Plus: the browser devtools **Network tab** (watch TaskFlow's frontend talk to its API —
every fetch, header, and status is visible), `python3 -m http.server` (instant local
server for experiments), and `dig`/`ping`/`lsof -i :3000` when things get weird.

## 8. Lab Track — Where to Practice, In Order

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | With TaskFlow running (`npm start`), replay its README curl examples with `-v`; find the same calls in the browser Network tab; trigger and read a 422 and a 404 | TaskFlow | HTTP anatomy on a server you own |
| 2 | Read `public/js/api.js` and `server/middleware/errorHandler.js` as a pair | TaskFlow | Both ends of one contract; errors at the boundary |
| 3 | **Story S3-1 (specced in Ledger's backlog):** exchange-rate client for the free Frankfurter API — adapter module, explicit timeouts, backoff-with-jitter retries, local TTL cache, offline fallback, failure-path tests | Ledger | The full client discipline of §3, in stdlib Python |
| 4 | **Story S3-1 (specced in TaskFlow's backlog):** import tasks from a GitHub repo's issues — token auth from env, `Link`-header pagination, rate-limit handling, external→domain mapping | TaskFlow | Real-world API integration against a big public API |
| 5 | Icebox, later: TaskFlow WebSocket live updates; webhook receiver with signature verification | TaskFlow | Push-based integration |

**Research prompts:** what happens end-to-end when you hit Enter on a URL (the classic
interview question — now you have hooks for every stage); idempotency keys at Stripe;
why DNS has TTLs; connection pooling in HTTP clients; OpenAPI specs and generated
clients; gRPC vs REST trade-offs.
