# Network Programming — Sockets, Protocols & Concurrent Servers

Module 08 taught you to be a good *client* of an HTTP API — the application layer, where a
framework has already turned the network into tidy request/response objects. This module
goes underneath that, to the layer HTTP itself is built on: raw sockets, the byte stream,
and the machinery a server needs to talk to many clients at once. It's the "build the
primitive before you trust the abstraction" philosophy — the same reason this workspace
does vanilla JS before React and a from-scratch game before an engine — applied to the
network.

It pairs with **[Switchboard](../project-8-switchboard/)**, a concurrent key-value store
built from `socket()` up, with its own documented wire protocol. Every concept below is
something you can see running there.

---

## 1. The Socket — the Actual API Under Everything

A socket is a file descriptor you can read and write, where the other end is another
program — possibly on another continent. The Berkeley sockets API (1983, and still the
foundation) is a handful of calls, and a TCP server uses all of them:

```
socket()   → create an endpoint
bind()     → claim an address + port  ("I am 0.0.0.0:6380")
listen()   → mark it as accepting connections (with a backlog queue)
accept()   → block until a client connects; return a NEW socket for that client
recv()     → read bytes from a connected socket
send()     → write bytes to it
close()    → hang up
```

The one that surprises people: `accept()` returns a *new* socket per client. The
listening socket is a receptionist that only ever hands you connections; each conversation
happens on its own socket. That single fact is what makes the concurrency question in §4
exist at all.

`curl`, Express, your browser, `psql`, every networked program you've ever run bottoms out
in these calls. Frameworks hide them; they do not replace them.

## 2. The Lesson That Fixes Half of All Networking Bugs: TCP Is a Byte Stream

Here is the single most important thing in this module. **TCP does not preserve message
boundaries.** It is a stream of bytes, like a file. If the client sends `"HELLO"` then
`"WORLD"`, the server's `recv()` calls might return:

- `"HELLO"` then `"WORLD"` (what you naively expect), or
- `"HELLOWORLD"` in one call (the sends got coalesced — Nagle's algorithm, or just timing), or
- `"HEL"`, `"LOWOR"`, `"LD"` (split across calls — the data crossed packet boundaries), or
- any other chunking. `recv(4096)` means "give me *up to* 4096 bytes that have arrived,"
  not "give me the next message."

Code that assumes one `recv()` == one message works on localhost with small payloads and
then corrupts data in production. The bug is invisible in exactly the conditions you test
in. This is why the framing lesson has to be *felt*, and why Switchboard's tests
deliberately feed the parser bytes in cruel chunk boundaries.

### Framing: putting message boundaries back

Since TCP won't delimit messages, your *protocol* must. Two standard ways:

- **Length-prefixed** (Switchboard's choice): every message is `[4-byte length N][N bytes
  of payload]`. The reader reads exactly 4 bytes, decodes N, then reads exactly N more.
  Unambiguous, handles binary payloads containing any byte, and lets the reader know the
  full size up front (so it can reject a 4 GB frame before allocating for it — a security
  property, §5). The cost: you must know a message's length before you send it.
- **Delimiter-based**: messages end at a sentinel, classically `\r\n` (HTTP request lines,
  SMTP, Redis's RESP all do this). Simple and human-readable — you can `telnet` in and
  type — but the delimiter must be impossible in the payload, or you need escaping, and
  the reader must scan for the delimiter (buffering an unbounded amount if a malicious
  client never sends one).

The general skill: **the sender and receiver must agree, in advance, on how to find the
edges of a message in the stream.** That agreement is the first thing a wire protocol
specifies.

## 3. Designing a Wire Protocol

A protocol is an API contract at the byte level. Switchboard's `PROTOCOL.md` is a
first-class artifact for exactly this reason — the same weight module 08 gave a REST
contract. The decisions you make:

- **Text vs binary.** Text protocols (HTTP, SMTP, Redis RESP) are debuggable — you can
  read a packet capture and `telnet` in by hand — at the cost of size and parsing speed.
  Binary protocols (Protocol Buffers over gRPC, most databases' native wire formats) are
  compact and fast but opaque without tooling. Switchboard ships text-first for
  learnability and files "binary variant + benchmark" as an icebox story — the same
  debuggability-before-optimization call SearchLite made for its index format.
- **Request/response vs streaming vs push.** Switchboard is request/response (client asks,
  server answers, like HTTP). Other shapes: streaming (one request, many responses — think
  a subscription), and server push (the server speaks first or unprompted — the WebSocket
  model from TaskFlow's icebox).
- **Versioning from day one.** Put a version in the handshake or the frame. The day you
  need to change the protocol, old clients must still get a coherent error instead of
  silent corruption. This is API versioning (module 08 §2) at the transport layer.
- **Explicit errors.** The protocol must be able to *say* "bad request," "unknown
  command," "not found" — a vocabulary of failure, the byte-level cousin of HTTP status
  codes. A protocol with no error representation forces the server to hang up on problems,
  which is indistinguishable from a network fault to the client.

## 4. The Big Question: How Do You Serve Many Clients at Once?

One `accept()` gives you one client. A server needs hundreds. How you handle that is *the*
architectural decision of network programming, and every model is a different answer to
"what do we do while one client is slow?"

| Model | How it works | Strength | Weakness | In the wild |
|---|---|---|---|---|
| **Thread (or process) per connection** | `accept()`, hand the socket to a new thread, loop | Simplest correct model; blocking code reads naturally; uses multiple cores | Threads cost memory + scheduling; 10k threads is painful; shared state needs locks | Apache (prefork/worker); **Switchboard Sprint 1** |
| **Event loop / multiplexing** | One thread, `select()`/`poll()`/`epoll`/`kqueue` watches all sockets, handle whichever is ready | Handles 10k+ idle connections cheaply; no lock (single thread) | One slow handler blocks *everyone*; you must never block; inverted control flow | nginx, Node.js, Redis; **Switchboard Sprint 2** |
| **Thread pool** | Fixed N worker threads pull connections from a queue | Bounds resource use; predictable under load | Tuning N; a full pool queues or rejects | Most JVM servers, database connection pools |
| **Async / coroutines** | Event loop underneath, but `async`/`await` makes the code look blocking | Event-loop scale with readable code | The "function color" problem; a stray blocking call stalls the loop | Go (goroutines hide it entirely), Python `asyncio`, Rust `tokio` |

The lesson isn't "one is best" — it's understanding the trade. The famous
**C10k problem** (can one server handle 10,000 simultaneous connections?) was what drove
the industry from thread-per-connection toward event loops in the 2000s; it's why nginx
displaced Apache for high-concurrency workloads and why Node's whole identity is a
single-threaded event loop. Switchboard has you build the thread-per-connection version
first (so you meet locking and races concretely — a shared store mutated by many threads),
then the `selectors` event-loop version (so you feel why "never block the loop" is law).
Building both is how the table above stops being trivia and becomes judgment.

## 5. Robustness — a Server Is a Public Target

The moment your program `listen()`s on a port, anything that can reach it can send it
anything. A server that assumes well-behaved clients is a server with an outage scheduled.
The defensive-programming from the practices doc (§7), at the network edge:

- **Bound everything.** Max frame length (Switchboard caps it — an attacker sending a
  4-byte length of `0xFFFFFFFF` must be refused, not handed 4 GB of allocation), max
  connections, max buffered bytes per connection. Every unbounded resource is a
  denial-of-service waiting to happen.
- **Timeouts on idle connections.** A client that connects and sends one byte an hour
  (the **Slowloris** attack) ties up a thread or a slot forever unless you time it out.
  Every socket read gets a deadline — the server-side mirror of the client timeouts in
  module 08 §3.
- **Validate at the boundary.** The protocol parser is a trust boundary as hard as any in
  this workspace: malformed frames, unknown commands, oversized payloads, invalid UTF-8 —
  all are *expected input from a hostile sender*, handled with a clean protocol error, never
  a crash. Switchboard's parser is tested with garbage on purpose.
- **Backpressure.** If a client reads responses slower than you produce them, your send
  buffer grows without bound. Real servers push back (block, drop, or disconnect) rather
  than run out of memory being "helpful."
- **Fail one connection, not the server.** An exception handling one client must never take
  down the accept loop. One bad conversation is not an outage.

## 6. Where This Sits Under Everything You've Built

This module is the floor beneath module 08's ceiling:

- **HTTP is a text, delimiter-framed, request/response protocol over TCP.** Everything
  TaskFlow's Express server does — parse a request, route it, write a response — is this
  module's machinery with the sockets and framing hidden. After Switchboard, read one raw
  HTTP request (`printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\n' | nc host 80`) and you'll see
  Express's whole job.
- **The repository pattern shows up again.** Switchboard's store sits behind an interface,
  network-unaware, exactly like Ledger's and TaskFlow's repositories — so it's testable
  without a socket and swappable (in-memory now; a persistent store is an icebox story).
- **TLS/HTTPS** is this, wrapped. A TLS socket is a normal socket with an encryption layer
  spliced in after a handshake; the framing and protocol above it are unchanged. That's why
  "add TLS" is a contained icebox story, not a rewrite.
- **The database module's connection pooling** is the client side of everything here — why
  opening a socket per query is expensive, and why pools exist.

## 7. Lab Track

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Read `protocol/` with the LEARNING_PATH — the codec and its framing tests, before any sockets | Switchboard | Framing as a pure, testable problem, isolated from I/O |
| 2 | Run the server; talk to it by hand with `nc` and a few `printf`ed frames; watch a `recv` log show messages arriving split and coalesced | Switchboard | The byte-stream reality (§2), observed live |
| 3 | **Break framing on purpose**: make the reader assume one `recv` == one message, then send a large value and watch it corrupt; revert | Switchboard | Why §2 is the most important section here |
| 4 | Sprint 2: build the `selectors` event-loop server behind the same protocol and store; load-test both models with many connections | Switchboard | The concurrency trade (§4) in your own hands |
| 5 | Sprint 3 icebox: reimplement the server in Go (`net` + goroutines) and write the comparison ADR — what the language gave you vs. what it hid | Switchboard | Appreciating the abstraction, having built the primitive |

**Research prompts:** the C10k problem and the move to `epoll`/`kqueue`; Nagle's algorithm
and `TCP_NODELAY` (why your small messages get coalesced); `SO_REUSEADDR` (why a
just-killed server can't rebind for a minute); the TCP three-way handshake and what
`listen()`'s backlog actually queues; head-of-line blocking (and how HTTP/2 and QUIC/HTTP3
attack it); Redis's RESP protocol (a beautifully simple real-world wire format to read);
why Go and its goroutine model became the default for network services.
