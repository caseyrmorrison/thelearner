# Algorithms in Practice — Where Classic CS Shows Up at Work

You've studied these algorithms. This doc maps them to the places they earn a salary —
specifically, where each one lives in the four projects and *why it was chosen over the
alternatives*. That second part is the industry skill: the choosing, not the coding.

---

## 1. The Honest Picture

Day-to-day product engineering is ~90% "use the hash map, use the standard sort, keep it
O(n log n) or better." The remaining 10% — knowing *when* you're in a situation that
needs a real algorithm, and recognizing which one — is a large part of what separates
senior engineers. These projects plant real 10%-situations in front of you.

Big-O in production terms: n = 100 → almost anything works; n = 10⁶ → O(n²) is an outage;
"it was fast in dev" usually means "dev had n = 12." The skill is asking *what is n in
production, and what happens to it next year?*

## 2. Algorithm Map

### TaskFlow (web) — Topological Sort & Cycle Detection
Tasks can declare dependencies ("deploy" blocks on "review"). Two real needs fall out:

- **Suggested work order** = topological sort of the dependency DAG. Implemented with
  **Kahn's algorithm** (repeatedly take a node with in-degree 0). Chosen over DFS-based
  toposort because Kahn's naturally reports *which tasks form a cycle* when one exists —
  and the API needs that for a useful error message. Same complexity, better diagnostics:
  a very typical real-world tiebreaker.
- **Cycle rejection** — the API must refuse a dependency edge that would create a cycle
  (A blocks B blocks A = nobody ever works). Detecting it *at write time* keeps the
  invariant "the graph is always a DAG" true forever, which is cheaper than handling
  cycles everywhere downstream. Protecting invariants at the boundary is an
  architecture-meets-algorithms decision.

Where you'll meet this again: build systems (make, npm, cargo), spreadsheet formula
evaluation, package managers, migration orderings, course prerequisites.

### Ledger (Python) — Aggregation, Sorting with Keys, and Money Arithmetic
- **Hash-map aggregation**: monthly spend per category = one pass building
  `dict[(month, category)] -> total`. The single most-used "algorithm" in industry.
  O(n), and the shape of 80% of all reporting code and every GROUP BY you'll ever write.
- **Decimal, not float** for money. `0.1 + 0.2 != 0.3` in binary floating point; in a
  finance app that's not trivia, it's a lawsuit. `decimal.Decimal` (or integer cents) is
  the professional norm. This is an *algorithmic correctness* decision invisible until
  it isn't — research "IEEE 754" and "float money bugs."
- **Moving average** over monthly totals for a spending trend — the simplest useful
  forecasting tool, and an intro to the smoothing-vs-lag trade-off that all of
  time-series analysis riffs on.
- **Sorting with key functions** rather than hand-rolled comparators, and *stable* sort
  as a feature: sort by amount, then by date, and equal dates keep the amount order.
  Python's sort (Timsort) is stable by design — knowing your stdlib's guarantees is the
  applied version of knowing the algorithms.

### Stockroom (Java) — Priority Queues, Greedy Allocation, Strategy Selection
- **Priority queue** (`java.util.PriorityQueue`, a binary heap) drives restock
  suggestions: always surface the most-urgent shortage next. Chosen over "sort the whole
  list every time" because urgency changes incrementally — heaps give O(log n) updates
  vs O(n log n) full re-sorts. The *access pattern* picked the data structure.
- **Greedy allocation** for filling orders from limited stock. Greedy is correct here
  because allocations are independent — and the code comments spell out when greedy
  *breaks* (bin-packing-like coupling between choices), because knowing an algorithm's
  failure conditions is the difference between using it and gambling with it.
- **Strategy pattern for pricing** is an algorithms decision wearing an OOP costume:
  selecting among interchangeable algorithms (flat discount, bulk tiers, none) at
  runtime. Big-O identical; changeability is the axis being optimized.

### SearchLite (C++) — Inverted Index, TF-IDF, Top-K with a Heap
- **Inverted index**: `term -> [(document, positions)]`. *The* data structure of search
  (Google, Elasticsearch, your IDE's symbol search). Turns "scan every document per
  query" O(total text) into O(matching postings). Space traded for query time — the
  classic systems trade.
- **TF-IDF ranking**: term frequency × inverse document frequency. A word matters if
  it's frequent *in this document* and rare *across documents*. Simple, decades old,
  still the baseline modern search (BM25, embeddings) is measured against. Research
  path: TF-IDF → BM25 → vector/embedding search.
- **Top-K selection**: users want 10 results, not all 40,000 scored docs sorted. A
  bounded min-heap of size K gives O(n log K) instead of O(n log n) — trivial-looking,
  and one of the most commonly *missed* optimizations in real code ("just sort it" is
  the reflex; K=10 vs n=40,000 says otherwise).
- **Hashing under the hood**: `unordered_map` powers the index; the comments discuss
  load factors and *why iteration order isn't stable* — a production bug generator.

## 3. The Meta-Skill: Choosing

The pattern in every case above:

1. **Name the operation that dominates** (writes? reads? updates? top-k queries?).
2. **Estimate n, now and in two years.** Small n → simplest code wins, full stop.
3. **Know what each candidate is bad at** — you pick data structures by their weaknesses
   (hash map: no order; heap: no random access; array: no cheap middle-insert).
4. **Write the why down** where the next person will look (comment or ADR).

Interviews test step 3 in isolation. Jobs pay for all four together.

## 4. Research Prompts

After meeting each algorithm in code, go one ring outward:

- Kahn's vs DFS toposort; incremental cycle detection (adding edges to a live DAG)
- Timsort — why real-world sorts exploit existing runs in data
- BM25, and why raw TF-IDF over-rewards long documents
- Bloom filters — where a "maybe" answer is worth 10× memory savings
- Consistent hashing — the algorithm that makes distributed caches possible
- LRU caches (hash map + doubly-linked list) — a beautiful composite structure and a
  perennial interview favorite; a natural Sprint-3 story for SearchLite
