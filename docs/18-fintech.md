# FinTech — Money, and the Software That Moves It

A from-zero tour of financial technology for an engineer who has never worked in it. The
promise: by the end you'll understand what the field *is*, the concepts that recur in every
corner of it, and — because this is an engineering handbook — why fintech is one of the most
demanding domains software engineers work in. No finance background assumed; every term is
defined. But you already have a head start you may not realize: **Ledger** (project 1) taught
you to handle money correctly, and that discipline is the foundation everything here stands on.

It pairs with **[Tally](../project-10-tally/)**, a double-entry ledger engine — §2's core
concept, built so its central rule (the books must always balance) is a test that runs.

---

## 1. What FinTech Actually Is

FinTech is technology reshaping financial services — but the useful engineer's framing is
sharper: **a fintech company is a software company that happens to move, store, or account
for money.** That "happens to" hides everything hard. When your bug is a broken button,
you ship a fix. When your bug is money in the wrong place, you have a financial loss, an
angry customer, and possibly a regulator. The engineering bar — correctness, security,
auditability — is set by that difference.

The landscape, so the jargon has a home:

- **Payments** — moving money between parties (Stripe, PayPal, Square, Adyen). The busiest
  fintech sector and where most engineers start.
- **Banking / neobanks** — software-first banking (Chime, Revolut, Monzo). Usually *not*
  banks themselves — see §5.
- **Lending / credit** — loans, credit, "buy now pay later" (Affirm, Klarna). Risk and
  math heavy.
- **Wealth / investing** — brokerages and robo-advisors (Robinhood, Wealthfront).
- **Insurtech** — insurance, reimagined with data.
- **Crypto / blockchain** — decentralized ledgers and assets (§8).
- **Infrastructure ("picks and shovels")** — the APIs other fintechs build on (Stripe,
  Plaid, Marqeta). Often the best engineering, because their product *is* an API.

The through-line: every one of these is, underneath, a system that tracks who has how much
money. Which is a ledger. Which is §2.

## 2. The Foundational Idea: Money Is a Ledger

Here is the single most important concept in fintech, and it's 500 years old. In 1494 the
Venetian friar Luca Pacioli codified **double-entry bookkeeping**, the technique Venetian
merchants used to track trade. Every fintech system ever built is a descendant of it.

The idea: money never appears or vanishes — it *moves*. So every transaction is recorded in
(at least) two places: a **debit** to one account and an equal **credit** to another. Money
leaving here is money arriving there. The core rule:

> In every transaction, total debits equal total credits.

Accounts come in types, and the master equation they satisfy is the **fundamental
accounting equation**:

$$\text{Assets} = \text{Liabilities} + \text{Equity}$$

- **Assets** — what you own (cash, receivables).
- **Liabilities** — what you owe (customer balances — a bank's deposits are its liabilities!).
- **Equity** — the residual, what's left for the owners.
- Two flow types feed equity over time: **Revenue** (increases it) and **Expenses**
  (decrease it).

"Debit" and "credit" are *not* "add" and "subtract" — they're directions, and whether a
debit increases or decreases an account depends on the account's type (its "normal
balance"). That trips up every newcomer; Tally's code makes the rule explicit.

**Why this 500-year-old technique is the bedrock of modern fintech:**

- **Money is conserved.** Because debits must equal credits, money can't be created or
  destroyed by a bug — only misrouted, which is detectable.
- **Errors are detectable.** Sum every debit and every credit across the whole system; they
  must be equal (the **trial balance**). If they're not, something is wrong, and you know it
  *before* a customer does. That's a system-wide correctness invariant you get for free.
- **It's an audit trail.** The ledger is append-only — you never edit or delete a
  transaction; you post a *reversing* entry. History is immutable, which is exactly what
  auditors and regulators require (and what Ledger's soft-delete icebox story gestured at).

This is why Tally exists: to make the balancing rule something you *feel* as an invariant,
not just read about. It's the fintech equivalent of implementing backprop by hand.

## 3. Money Representation — Where Your Ledger Knowledge Pays Off

You already learned this, and in fintech it stops being a nicety and becomes the law:

- **Never floating point.** `0.1 + 0.2 ≠ 0.3` in binary floats; in a payments system that's
  not a rounding curiosity, it's a reconciliation break and a compliance finding. Use
  integer **minor units** (cents, pence, satoshis) or arbitrary-precision decimals. This is
  Ledger's [ADR-0002](../project-1-ledger-cli/docs/adr/0002-money-as-integer-cents.md), and
  Tally inherits it.
- **Currency is part of the amount.** `100` is meaningless; `100 USD` is money. A `Money`
  type carries both (Stockroom modeled this too). Mixing currencies in arithmetic must be
  impossible, not merely discouraged.
- **Rounding is a policy you state out loud.** When you split a $10.00 fee three ways,
  someone gets $3.34 and the math must still sum to exactly $10.00 — "banker's rounding"
  and explicit remainder allocation exist for this. Silent rounding is how a ledger drifts
  by a cent a day until it's a cent times millions.

The lesson: the correctness discipline that felt like over-engineering in a personal expense
tracker is the *minimum viable standard* in fintech.

## 4. Payments — How Money Actually Moves

Naively, a payment is "money goes from A to B." Really it's a multi-party, multi-day state
machine, and understanding it is most of payments engineering.

### The card model (the four-party dance)
When a card is charged, at least five parties are involved:

1. **Cardholder** — the customer.
2. **Merchant** — you.
3. **Acquirer** — the merchant's bank/processor.
4. **Issuer** — the cardholder's bank.
5. **Card network** — Visa/Mastercard, the rails in the middle.

And a "payment" is several distinct steps, often spread over days:

- **Authorization** — the issuer checks funds and places a hold. No money has moved.
- **Capture** — the merchant claims the authorized amount (can be later, or less — think
  hotels, tips).
- **Clearing & settlement** — the networks and banks actually move funds, net out what each
  party owes, and deposit to the merchant, minus fees. Days later.
- **Interchange fee** — a cut (a couple percent) the issuer takes, the core economics of the
  card industry.

So "the payment succeeded" (authorized) and "you have the money" (settled) are different
events, sometimes days apart. Model a payment as an explicit **state machine**
(`pending → authorized → captured → settled`, with `failed`, `refunded`, `disputed`
branches) — the same discipline as Rebound's game states, applied to money.

### Bank rails (moving money without cards)
- **ACH** (US) — batched, 1–3 days, cheap, reversible. Payroll, bill pay.
- **Wire** — fast, expensive, effectively irreversible. Big one-off transfers.
- **RTP / FedNow** (US), **SEPA Instant** (EU), **UPI** (India) — instant rails, the modern
  direction.
- **Card networks** — fast for the customer, but settlement lags (above).

Speed, cost, and reversibility trade off against each other; choosing a rail is an
engineering-and-product decision.

### Idempotency — the payments non-negotiable
If a "charge card" request times out, did it go through? You must be able to safely retry
*without* double-charging. Every serious payments API uses **idempotency keys**: a
client-generated unique id per logical operation; the server processes the first, and
returns the *same result* for any retry with that key. You met this in the
[networking module](08-networking-and-apis.md) and the capstone — in fintech it's not an
optimization, it's the line between correct and catastrophic. Tally posts entries
idempotently for exactly this reason.

**Stripe** is the canonical developer-facing payments API and the reference implementation
of most ideas here (idempotency keys, the payment state machine, webhooks for async
settlement events). When you build a real integration, this environment has a
**`stripe-best-practices` skill** that encodes the current right way to do it — reach for it
then rather than guessing from memory.

## 5. Banking, Identity, and Compliance — the Part That Shapes Architecture

**You are almost never a bank.** Chartering a bank is a years-long regulatory process, so
most "neobanks" are software front-ends on a **sponsor bank** that holds the actual charter
and deposits. **Banking-as-a-Service (BaaS)** providers expose a chartered bank's
capabilities as APIs. The engineering consequence: your ledger (§2) must **reconcile** (§9)
against the sponsor bank's ledger, continuously — two sources of truth that must agree.

**Compliance is not a footnote; it's an architectural force:**

- **KYC** ("Know Your Customer") — verifying identity before onboarding. It's a required
  gate in your signup flow, not a nice-to-have.
- **AML** ("Anti-Money-Laundering") — monitoring transactions for suspicious patterns,
  screening against sanctions lists. This is a real-time data pipeline you must build.
- **PCI-DSS** — the security standard for handling card data. The winning move is usually to
  *never touch raw card numbers* (let Stripe tokenize them), shrinking your compliance scope
  dramatically. Architecture as compliance strategy.

**Data aggregation** (Plaid, and open-banking APIs) lets apps connect to users' existing
bank accounts to read balances and transactions — the plumbing behind "link your bank
account."

The meta-point: in most domains, regulation constrains what you build; in fintech it
constrains *how you build it* — your data model, your flows, your audit logs. Engineers who
treat compliance as someone else's problem write architectures that have to be torn up.

## 6. Lending & Credit — Risk as a Product

Lending is renting money, and the engineering is about **risk** and **math**:

- **Underwriting** — deciding whether to lend and at what price, by assessing default risk.
  Increasingly this is a machine-learning problem — which connects straight to the
  [ML module](17-machine-learning.md): a credit model is a classifier predicting default,
  trained on borrower features. But it's ML under *legal constraints* most ML isn't: in the
  US, credit decisions must be explainable (adverse-action notices) and non-discriminatory
  (fair-lending law), which rules out black-box models you can't interrogate. Fintech is
  where the ML fairness and interpretability topics stop being academic.
- **Interest math** — worth two formulas. **APR** (annual percentage rate) is the simple
  annualized rate; **APY/EAR** accounts for compounding:
  $\text{APY} = \left(1 + \frac{r}{n}\right)^{n} - 1$ for rate $r$ compounded $n$ times a
  year. And an **amortizing loan**'s fixed payment is

  $$M = P\,\frac{i(1+i)^{n}}{(1+i)^{n} - 1}$$

  where $P$ is principal, $i$ the periodic rate, $n$ the number of payments — the formula
  behind every mortgage and car-loan schedule. Each payment splits into interest (on the
  remaining balance) and principal; early payments are mostly interest. Building an
  amortization schedule is a great fintech exercise (and a Tally backlog story).

## 7. Investing & Market Infrastructure (Briefly)

Brokerages and exchanges match buyers and sellers via an **order book** — bids and asks
sorted by price, matched by a **matching engine** (a lovely data-structures problem: price-
time priority, often heaps/trees — echoing SearchLite's and Stockroom's priority-queue
work). After a trade, **clearing and settlement** actually exchange cash for the asset,
historically days later ("T+1" — trade date plus one). **Robo-advisors** automate portfolio
allocation. The recurring engineering themes are the same ones you've met: correctness,
ordering guarantees, and reconciliation.

## 8. Crypto & Blockchain — the Trustless Ledger

Cut through the hype to the engineering idea, and a blockchain is: **a replicated, append-
only, cryptographically-verified ledger with no central owner.** Look back at §2 — it's
double-entry bookkeeping's trustless cousin. Where a bank's ledger is trusted because the
bank is regulated, a blockchain's ledger is trusted because thousands of nodes each hold a
copy and a **consensus** mechanism (proof-of-work, proof-of-stake) makes rewriting history
computationally or economically infeasible.

What it genuinely provides: no trusted intermediary, censorship-resistance, programmable
money (**smart contracts**). What it costs: throughput, latency, energy (PoW), and a UX and
regulatory story still being written. **Stablecoins** (tokens pegged to a currency) are the
part most relevant to mainstream payments today. Hold both truths: the core CS is real and
elegant, and most of the surrounding market is noise. Evaluate specific claims on their
engineering merits.

## 9. Why FinTech Is Hard Mode for Engineers

This section is the point of the module. Nearly every discipline in this handbook shows up
here with the stakes turned up, because the bug costs money:

- **Correctness is conserved money.** The double-entry invariant (§2) means errors are
  detectable — *if* you build on a real ledger. Tally makes this an executable test.
- **Idempotency & exactly-once.** Process a payment twice and you've moved money twice.
  Idempotency keys and deduplication aren't features, they're survival (§4).
- **Auditability & immutability.** You never delete a financial record. Corrections are
  reversing entries; the ledger is append-only. Regulators, auditors, and your own
  debugging all depend on it — and it makes **event sourcing** (store the events, derive the
  state) a natural fit.
- **Reconciliation.** The daily heartbeat of every fintech ops team: prove your ledger
  matches the bank's, the processor's, the card network's — independent sources of truth
  that must agree to the cent. When they don't ("a break"), someone investigates before
  close of business. Design for reconciliation from day one.
- **Consistency under distribution.** "Debit A, credit B" across services is the
  [capstone's](10-capstone-procurement-pipeline.md) dual-write problem in its highest-stakes
  form. This is why fintech leans on transactions, sagas, event sourcing, and idempotency so
  heavily.
- **Security & compliance as architecture (§5).** You are a target and you are regulated;
  both shape the design, not just the ops.

The encouraging flip side: **you have already practiced the foundations.** Money math
(Ledger), the trust boundary and validation (Switchboard, TaskFlow), idempotency
(networking, the capstone), auditability (Ledger's soft-delete), immutable decision records
(ADRs everywhere), and making illegal states unrepresentable (Stockroom's sealed types —
which is exactly how Tally forbids an unbalanced entry). FinTech is where all of it converges
and *matters most*.

## 10. Lab Track

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Re-read Ledger's money handling (`models.py`, ADR-0002) through a fintech lens | Ledger | The correctness floor, now with stakes |
| 2 | Read Tally's `JournalEntry` — an unbalanced entry can't be constructed; find where debits==credits is enforced | Tally | Double-entry as an executable invariant (§2) |
| 3 | Run Tally's demo: post a deposit, a payment-with-fee, a transfer, a refund; read the trial balance and confirm it's zero | Tally | Money conservation, seen (§2, §9) |
| 4 | Read the property tests: whatever sequence of entries you post, the trial balance stays balanced and Assets = Liabilities + Equity | Tally | Correctness invariants as tests (§9) |
| 5 | Sprint 2: build financial statements (balance sheet, income statement) *from* the ledger; then an amortization schedule from §6's formula | Tally | Deriving reports from the source of truth |
| 6 | Sprint 3 / real integration: wire a Stripe payment that posts to the ledger on the settlement webhook — invoke the **`stripe-best-practices` skill** | Tally + Stripe | Payments (§4) end to end, done right |

**Research prompts:** double-entry vs. single-entry (and why Ledger is the latter); the
payment state machine and webhooks; idempotency keys (Stripe's docs are the reference);
ledger-as-a-service (Modern Treasury, TigerBeetle — a database built *only* for double-entry,
worth studying); event sourcing and CQRS for financial systems; reconciliation strategies;
ISO 20022 (the emerging global payments message standard); how ACH files (NACHA format)
actually look; PCI-DSS scope reduction via tokenization; fair-lending constraints on ML
credit models.
