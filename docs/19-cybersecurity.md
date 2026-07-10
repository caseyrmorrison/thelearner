# Cybersecurity — Building Systems That Resist Attack

Security isn't a feature you bolt on; it's a property of everything you build. This module
is deliberately **defensive and educational**: you learn how attacks work *so you can stop
them*, the way a locksmith studies lock-picking to build better locks. The framing matters —
§8 covers the ethics and the law explicitly, because in security, authorization is the line
between a professional and a criminal.

You have a head start. Across this workspace you already built real defenses without always
naming them: parameterized SQL in Ledger, `textContent` output encoding in TaskFlow, a
bounded trust-boundary parser in Switchboard, secrets-kept-out-of-git everywhere. This module
gives those instincts a systematic name, and pairs with **[Aegis](../project-11-aegis/)**, a
defensive crypto/auth primitives library where "hash a password correctly" and "verify a 2FA
code" become code you can read and test against published vectors.

---

## 1. The Mindset — Security Is a Property, Not a Feature

Three ideas reframe how you build once security is in view:

- **The CIA triad** — every security goal is **Confidentiality** (only the right people can
  read it), **Integrity** (nobody can tamper undetected), or **Availability** (it stays up
  under stress). When you're unsure "is this a security issue?", ask which of the three an
  attacker could violate.
- **The attacker's mindset, used by defenders.** Attackers don't follow your happy path;
  they ask "what did the developer *assume*, and what happens if I break that assumption?"
  Empty input, a 4 GB payload, a negative quantity, a `'; DROP TABLE`, two requests racing.
  Every assumption is an attack surface. You've met this as "validate at the trust boundary" —
  Switchboard's parser treats every byte as hostile until proven safe. That *is* the security
  mindset.
- **Assume breach, and defend in depth.** Perfect prevention is impossible, so layer
  defenses: even if one fails, another catches it. A parameterized query *and* least-privilege
  DB credentials *and* input validation *and* monitoring. Related principle: **least
  privilege** — every component gets the minimum access it needs, so a compromise is
  contained. (Ledger's SQLite user needs no network; Switchboard binds to localhost.)

The uncomfortable truth: the defender must be right everywhere, the attacker right once. That
asymmetry is why the disciplines below are systematic rather than ad hoc.

## 2. Threat Modeling — Reasoning About What Can Go Wrong

Before defending, enumerate what you're defending and against whom. Threat modeling is that
enumeration, and it's a design activity, not a scan:

1. **What are the assets?** Data (user PII, money, credentials), availability, reputation.
2. **What's the attack surface?** Every input, endpoint, dependency, and trust boundary —
   every place untrusted data or actors touch your system. Draw the data-flow and mark where
   trust changes (the network edge, the DB, a third-party API). Those boundaries are exactly
   the ones this workspace already draws.
3. **What are the threats?** A useful checklist is **STRIDE**: **S**poofing (pretending to be
   someone), **T**ampering (altering data), **R**epudiation (denying an action), **I**nformation
   disclosure, **D**enial of service, **E**levation of privilege. Walk each asset against each.
4. **What are the mitigations, and what's the residual risk?** You won't fix everything;
   decide explicitly what you accept — an ADR-shaped decision.

The payoff: security bugs are cheapest to fix at design time. A threat model is to security
what a test plan is to correctness — and both are documents a good team actually writes.

## 3. The Vulnerability Classes Every Engineer Must Know

These recur across all software; the industry catalog is the **OWASP Top 10** (web) and CWE
(the broader dictionary). Each below is paired with the defense — several of which you've
already built.

- **Injection** (SQL, command, LDAP…). Root cause: mixing *code* and *data* so attacker data
  becomes executable ("`'; DROP TABLE students;--`"). Defense: never concatenate untrusted
  input into a command; use **parameterized queries** (Ledger does this — study
  `repository.py`) and safe APIs that separate code from data. The general principle beyond
  SQL: keep the interpreter from ever seeing attacker-controlled structure.
- **Cross-Site Scripting (XSS)**. Attacker injects script into a page other users load.
  Defense: **output encoding** — treat all user data as text, never markup. TaskFlow's rule
  "user data only ever flows through `textContent`, never `innerHTML`" is exactly this, plus
  a **Content-Security-Policy** header as defense-in-depth.
- **Broken authentication / session management.** Weak passwords, guessable tokens, sessions
  that don't expire. Defense: proper password hashing, high-entropy tokens, MFA — all in
  §5 and Aegis.
- **Broken access control** (authorization). The most common serious web bug. **IDOR**
  (Insecure Direct Object Reference): the server trusts a client-supplied id
  (`GET /account/1234`) without checking *this user* may see it — the "confused deputy." A
  valid session (authn) is not permission (authz); check ownership on every access. TaskFlow's
  "never trust client state over server truth" is the seed of this.
- **Cryptographic failures.** Plaintext secrets, weak/rolled-your-own crypto, hashing
  passwords with fast hashes. Defense: §4.
- **Security misconfiguration** — default credentials, verbose errors leaking internals, an
  S3 bucket set public. Defense: secure defaults, least privilege, and *don't leak stack
  traces to users* (Ledger/TaskFlow translate errors at the boundary — a security property,
  not just tidiness).
- **Vulnerable dependencies & supply chain** — most of your code is other people's. Defense:
  §6.
- **SSRF, CSRF, insecure deserialization** — know the names: SSRF tricks your server into
  making requests it shouldn't (fetch a URL an attacker controls → internal network);
  CSRF rides a logged-in user's cookies to act as them (defense: anti-CSRF tokens, SameSite
  cookies); insecure deserialization turns "load this saved object" into code execution
  (never deserialize untrusted data with a format that can instantiate arbitrary types —
  a reason `pickle` is dangerous and JSON is safer).

You don't memorize exploits; you internalize the *pattern* — untrusted input reaching a
powerful sink — and the *defense* — validate on the way in, encode on the way out, least
privilege everywhere.

## 4. Cryptography for Engineers — Use It, Don't Invent It

The first rule of crypto engineering: **don't roll your own.** Not the algorithms — those are
designed and broken by specialists over decades. Your job is to *compose vetted primitives
correctly*, which is itself easy to get wrong. What each primitive gives you:

- **Encoding ≠ encryption ≠ hashing** — the classic confusion. **Encoding** (Base64) is
  reversible and provides *nothing* secret; it's for transport. **Hashing** is one-way (no
  key, can't reverse) — for integrity and password storage. **Encryption** is reversible
  *with a key* — for confidentiality. Calling Base64 "encryption" is a career-limiting move.
- **Symmetric encryption** (AES) — one shared key encrypts and decrypts. Fast; the problem is
  sharing the key. Use an **authenticated** mode (AES-GCM) so tampering is detected — never
  raw ECB (it leaks patterns — search "ECB penguin").
- **Asymmetric / public-key** (RSA, elliptic curve) — a public key encrypts / verifies, a
  private key decrypts / signs. Solves key distribution; underlies TLS and signatures. Slow,
  so it's used to bootstrap a symmetric key (which is what a TLS handshake does — recall the
  [networking module](16-network-programming.md)).
- **Hash functions** (SHA-256) — fixed-size one-way fingerprints; the integrity workhorse.
  But **not** for passwords directly (§5).
- **KDFs / password hashing** (scrypt, argon2, bcrypt, PBKDF2) — *deliberately slow* hashes
  with a salt and a work factor, for storing passwords. Slowness is the feature. Aegis
  implements this with stdlib `scrypt`.
- **MACs / HMAC** — a keyed hash proving a message is authentic *and* untampered. Aegis
  implements HMAC and shows why verification must be **constant-time** (§5).
- **Digital signatures** — asymmetric integrity + non-repudiation (only the private-key holder
  could have signed; anyone can verify). The basis of software signing, certificates, JWTs.

TLS ties these together: it's authenticated key exchange (asymmetric) bootstrapping a fast
encrypted channel (symmetric) with integrity (MACs). "Add HTTPS" is composing this correctly,
which is why you use a library, never a hand-rolled socket-crypto stack.

## 5. Authentication & Authorization — Proving Who, Deciding What

Two different questions engineers constantly conflate:

- **Authentication (authn)** — *who are you?* Passwords, tokens, keys, biometrics.
- **Authorization (authz)** — *what may you do?* Checked on every sensitive action, because a
  valid identity is not blanket permission (the IDOR lesson).

**Password storage, done right** (Aegis builds this):
- Never store plaintext. Never store a plain `sha256(password)` either — fast hashes fall to
  **rainbow tables** and GPUs cracking billions/second.
- **Salt** every password with a unique random value, so identical passwords hash differently
  and precomputed tables are useless.
- Use a **slow KDF** (scrypt/argon2/bcrypt) with a tuned **work factor**, so each guess is
  expensive at scale but a single login stays fast enough.
- **Verify in constant time.** A naive `==` on secrets leaks *where* the first difference is
  via timing, and attackers measure it (a **timing attack**). Use `hmac.compare_digest`. Aegis
  demonstrates the timing difference so the abstract threat becomes concrete.

**Beyond passwords:**
- **MFA / 2FA** — "something you know" + "something you have." **TOTP** (RFC 6238) is the
  authenticator-app standard: client and server share a secret and both compute an HMAC of the
  current 30-second time window. Aegis implements TOTP from scratch and verifies it against the
  RFC's own test vectors — and you can scan it into a real authenticator app.
- **Sessions vs tokens (JWT)** — server-side sessions (a random id indexing server state) vs
  self-contained signed tokens (recall networking §4); the trade is revocability vs.
  statelessness.
- **OAuth 2.0** — delegated authorization ("Sign in with Google" without sharing your
  password); recall the networking module. Don't hand-roll it.
- **Secure tokens** — session ids, reset tokens, and API keys must come from a **CSPRNG**
  (`secrets`), never `random` (a predictable PRNG — predicting a "random" reset token is a
  real account-takeover bug). Aegis contrasts the two.

## 6. Secrets, Dependencies, and the Supply Chain

- **Secrets management.** Never in git (history is forever — recall the practices doc);
  never in code. Use environment variables, a secrets manager (Vault, cloud KMS), and
  **rotate** keys. A leaked key is an incident; design so rotation is easy.
- **The dependency attack surface.** Most of your shipped code is other people's. A malicious
  or compromised package runs with your privileges (the **supply-chain attack** — research
  "event-stream", "SolarWinds", "xz backdoor"). Defenses: pin versions with **lockfiles**
  (which is why `package-lock.json` is committed), audit (`npm audit`, `pip-audit`), minimize
  dependencies, and increasingly generate an **SBOM** (software bill of materials) so you know
  what you actually ship.
- **Least privilege for machines too** — CI tokens, deploy keys, and service accounts scoped
  to exactly what they need. The blast radius of any leak is whatever that credential could do.

## 7. Operations, Detection, and the Human Layer

Prevention fails eventually, so mature security also detects and responds:

- **Network & infra** — TLS everywhere (recall networking); firewalls and segmentation
  (least privilege for traffic); a **bastion**/jump host as a single hardened entry point.
- **Detection & response** — you can't stop what you can't see. Security logging (who did
  what, auth failures, privilege changes), monitoring, and **incident response** are the
  operational side of the [observability module](09-debugging-and-observability.md); the
  blameless post-mortem applies to breaches too.
- **The human layer is the real front line.** **Phishing** and **social engineering** cause
  more breaches than clever exploits — it's easier to trick a person than break AES.
  Technical defenses (MFA blunts stolen passwords) plus a culture where "verify unusual
  requests" is normal. Never underestimate this; the most sophisticated attacks often start
  with a convincing email.

## 8. Doing Security Ethically and Legally — Read This

Everything above is defensive knowledge, and the line between defender and attacker is
**authorization**, full stop:

- **Only test systems you own or have explicit written permission to test.** Probing someone
  else's system without authorization is a crime (the US Computer Fraud and Abuse Act, the UK
  Computer Misuse Act, and equivalents worldwide) — regardless of intent or whether you cause
  harm. "I was just curious" is not a defense.
- **Learn offense on purpose-built legal ground.** These exist precisely so you can practice
  attacking safely and legally: **PortSwigger Web Security Academy** (free, excellent),
  **TryHackMe** and **Hack The Box** (guided labs), **OverTheWire** (wargames), **PicoCTF**
  and other **CTF** competitions, and locally, deliberately-vulnerable apps you run on your
  own machine (OWASP Juice Shop, WebGoat). The system that hosts you here explicitly supports
  CTF and authorized-testing learning — this is the sanctioned path.
- **Responsible disclosure.** If you find a real vulnerability in someone's system, report it
  privately through their security contact or **bug-bounty** program and give them time to fix
  it before any public mention. Weaponizing or dumping it is where research becomes crime.
- **Professional paths** are real and in demand: security engineering, application security,
  penetration testing (authorized, contracted), incident response, security research. All are
  built on the defensive foundation this module teaches.

The knowledge is dual-use; the ethics are not. Build defenses, break only what you're
authorized to break, and disclose responsibly.

## 9. Lab Track

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Audit your own past code: find the trust boundaries in Ledger, TaskFlow, and Switchboard and name the attack each defense stops (parameterized SQL, `textContent`, bounded frames) | Prior projects | Security as a property you already built |
| 2 | Read Aegis's password module and its tests — salt, scrypt work factor, and `compare_digest`; then read the timing-attack demo | Aegis | Password storage done right (§5) |
| 3 | Run Aegis's TOTP against the RFC 6238 test vectors, then scan its secret into a real authenticator app and watch the codes match | Aegis | How 2FA actually works (§5) |
| 4 | Read the "insecure vs secure" contrasts (`sha256(pw)` vs salted-scrypt, `random` vs `secrets`, `==` vs `compare_digest`) — each with why the left column is a real bug class | Aegis | Secure-coding pattern recognition (§3–5) |
| 5 | Do a threat model (§2, STRIDE) of TaskFlow: list assets, walk the boundaries, note what's mitigated and what isn't | TaskFlow | Design-time security reasoning |
| 6 | Pick one PortSwigger Academy lab (e.g. SQL injection or access control) and solve it — legal, guided offense that cements the defense | PortSwigger (external) | Offense-for-defenders, ethically (§8) |

**Research prompts:** the OWASP Top 10 in depth; how argon2id beat scrypt/bcrypt (the password-
hashing competition); why AES-GCM over AES-CBC; the "ECB penguin"; certificate pinning and the
TLS trust model; how Signal's protocol achieves forward secrecy; the xz-utils backdoor
(2024) as a supply-chain case study; SameSite cookies and CSRF; capability-based vs ACL
security models; the CFAA and *Van Buren v. United States*; what a real bug-bounty report looks
like.
