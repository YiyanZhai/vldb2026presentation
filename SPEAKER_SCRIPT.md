# Clock2Q+ — 12-Minute Speaker Script (VLDB 2026)

Aligned slide-by-slide with `index.html`. Target ≈ **11:45 spoken**, leaving buffer for pacing.
Delivery: ~130 words/min, pause on each figure reveal. Advance fragments (→) as you reach the matching sentence.

---

## Slide 1 — Title · ~0:30
**Say:**
> "Hi everyone, I'm Yiyan Zhai from Carnegie Mellon — this is joint work with collaborators at Harvard and Broadcom. I'll talk about **Clock2Q+**, a cache replacement algorithm we built for *metadata* caches, and that's running in production in **VMware vSAN**. The idea is simple, so my goal is that by the end you'll understand exactly *why* it works."

**Remember:** A simple, production-deployed idea — set the expectation that it's intuitive.

---

## Slide 2 — The metadata cache is the hot path · ~1:00
**Transition:** "Let me start with where this lives."
**Say:**
> "Caches sit between fast memory and slow storage — keep the hot pages in DRAM, avoid going to SSD or disk. Now, vSAN keeps *two* kinds of cache: one for data, and one for metadata. And metadata is special, because metadata is what you consult to *find* the data in the first place. So the metadata cache is on the hottest path in the system — its replacement policy runs constantly, under heavy concurrency, often when the hit ratio is already near 100%. That changes what 'a good policy' means: it's not just miss ratio. A tiny per-hit cost, a lock bottleneck, or one rare corner case turns straight into lost throughput. So overhead, scalability, and *simplicity* matter just as much."

**Remember:** For a production metadata cache, miss ratio is necessary but not sufficient — overhead and simplicity are first-class.

---

## Slide 3 — Metadata has correlated references · ~1:15
**Transition:** "And metadata doesn't just run hotter — it *behaves* differently."
**Say:**
> "Here's the key pattern. A single high-level operation — say, walking or updating an index — touches many entries that happen to live on the *same* metadata page. So that one page gets hit several times in a quick burst… and then it goes quiet, maybe for a long time. We call these **correlated references**. And the important thing is: a burst like that is *weak* evidence of long-term popularity. It's one operation, not a hot block. This plot makes it concrete — on the left, metadata pages get *multiple* re-references inside a short window; on the right, data pages are mostly one-hit wonders. Data caches and metadata caches genuinely live in different worlds."

**Remember:** Correlated bursts *look* hot but aren't — this is the pattern the whole talk is about.

---

## Slide 4 — Why existing policies fall short · ~1:15
**Transition:** "So how do today's best policies handle a burst like that? There are two camps, and both stumble."
**Say:**
> "S3-FIFO — the recent state of the art — uses a small admission queue with a reference bit: re-hit a block while it's in that queue, the bit flips, and it gets promoted. But under a correlated burst, a *cold* page gets hit a few times near the head, its bit flips, and it's promoted into the main cache — where it just sits there as pollution. The older policies, 2Q and Clock2Q, go the opposite way: a genuinely hot block can only reach main *after* it's been evicted and then requested again — so every hot block pays one extra, avoidable miss. So we're stuck: filter too loosely and you admit junk; filter too aggressively and you miss real hot blocks. We want S3-FIFO's fast direct promotion *without* swallowing the bursts."

**Remember:** Over-promote (S3-FIFO) vs. under-promote (Clock2Q) — we want the best of both.

---

## Slide 5 — Key insight: the correlation window · ~1:00
**Transition:** "Here's the insight — and it's almost embarrassingly simple."
**Say:**
> "The realization is that *where* a re-hit happens tells you what it means. If a block gets re-hit right after it's inserted — near the head of the small queue — that's almost certainly part of the *same* burst, so we should ignore it. But if it's re-hit later, after it's aged a bit, that's real reuse worth acting on. So we add a **correlation window** at the head of the small queue: hits *inside* the window do **not** set the reference bit; hits *beyond* it do. Inside the window, bursts are filtered out. Beyond it, genuinely hot blocks still get promoted directly. And that's it — a one-line change, no new tuning knobs, no ordered LRU list."

**Remember:** *Where* the re-hit happens is the signal — one small change to S3-FIFO.

---

## Slide 6 — Clock2Q+ design · ~1:30
**Transition:** "Let me put that window into the full picture."
**Say:**
> "There are three queues. New blocks come into the **Small FIFO**, and its head portion is the correlation window. Now follow the paths. If a block reaches the tail *without* earning a reference bit — because it was cold, or it was just a burst inside the window — it ages out into the **Ghost** queue, which stores only the key, not the data. If instead it earned Ref = 1 *beyond* the window, it's promoted straight into the **Main Clock** — no extra miss. If a key that's sitting in Ghost gets requested again, that's proven reuse, so we admit it directly to Main. And the Main Clock itself uses the classic second-chance rule: the reference bit buys a block one more pass of the clock hand before it's evicted. For sizing, the Small FIFO is 10% of the cache, the Main Clock 90%, and Ghost is 50%."

**Remember:** Four clean paths — cold/burst → Ghost, hot → Main directly, Ghost-hit → Main, Main uses second-chance.

---

## Slide 7 — Clock2Q+ in action · ~1:30
**Transition:** "Rather than trace that on a static diagram, let's just watch it."
**Say (narrate over the animation as each scenario plays):**
> "Watch the small queue. A **cold** block ages to the tail with its bit still zero, and drops into Ghost. Now a **correlated burst** — it gets hit, but *inside the window*, so the bit stays zero; it also ages out to Ghost. That's the whole point: the burst never touches the Main Clock. Here's a **hot** block — it's hit *beyond* the window, so it earns Ref = 1, and when it reaches the tail it's promoted straight to Main. Now a **Ghost hit** — a key we remembered comes back, and we admit it directly to Main. And finally, in the **Main Clock**, a block with the bit set gets a second chance; the hand clears it and moves on, and evicts a cold one. The window is doing quiet work in the background while the fast path stays fast."

**Remember:** The window silently filters bursts; the direct-promotion fast path is untouched.
*(Let the loop finish one scenario set, then advance.)*

---

## Slide 8 — Evaluating metadata caches · ~1:00
**Transition:** "Now, how do you even evaluate a metadata policy? That turns out to be hard."
**Say:**
> "There are no public metadata-cache traces, and production traces can't be shared. So here's our trick: take *any* public block trace, and divide each block number by the B-tree fan-out — around 200. That reconstructs the sequence of leaf-page accesses a real metadata cache would see. We validated it against a real B-tree trace and the miss-ratio curves match to within one-hundredth of a percent. So now anyone can regenerate metadata-cache results from public data — it's fully reproducible. On top of that we ran 106 CloudPhysics traces — over two billion requests — against ten state-of-the-art policies."

**Remember:** A simple, validated recipe makes metadata-cache evaluation reproducible for everyone.

---

## Slide 9 — Miss ratio vs. 10 SOTA policies · ~1:15
**Transition:** "So — does the window actually pay off?"
**Say:**
> "These box plots show relative miss-ratio improvement, and we've put **Clock2Q+ as the zero line** — every other policy is measured against us. On metadata, on the left, every competitor's box sits *below* the line, meaning worse than Clock2Q+. On data traces, on the right, it's the same story. Across every cache size we tested, Clock2Q+ has the best **median and mean** improvement — and on some metadata traces we beat S3-FIFO, the strongest competitor, by up to **28.5%**. The headline isn't one lucky trace, though — it's that we win *across the board*, on both metadata and data."

**Remember:** Best median and mean at every cache size, on metadata *and* data.

---

## Slide 10 — Why it works · ~1:00
**Transition:** "Let me show you the mechanism behind that number."
**Say:**
> "This table counts how many blocks each policy promotes from the small queue into the Main Clock. S3-FIFO promotes about 88,000; Clock2Q+ promotes about 21,000 — under a quarter as many. We're far more selective, because the window is filtering the correlated bursts before they can be promoted. And it's not just fewer — it's the *right* ones: the blocks we promote have short reuse distances, so they're genuinely hot, while the ones we send to Ghost have long reuse distances. So the window filters more, and it filters correctly."

**Remember:** We promote under ¼ as many blocks — and they're the genuinely hot ones.

---

## Slide 11 — Engineered for production · ~0:45
**Transition:** "And because this actually ships in vSAN, miss ratio was never the only requirement."
**Say:**
> "Everything is array-based with no allocation, so the hit path is just a hash lookup and at most flipping one bit. It scales with fine-grained locks instead of a global one. It handles dirty pages by simply skipping them until they're flushed. It bounds the work per eviction, so there are no CPU spikes. And it resizes online. All of this came out of nearly a decade of evolving vSAN's cache — and throughout, simplicity was treated as a hard requirement, not a nice-to-have."

**Remember:** Low overhead, scalable, dirty-aware, bounded, resizable — simplicity by design.

---

## Slide 12 — Takeaways · ~0:45
**Transition:** "So, to wrap up."
**Say:**
> "Correlated references are a real and costly pattern in metadata caches, and they defeat the usual one-hit-wonder filters. The correlation window is a tiny change that fixes it — filter the bursts, but still promote truly hot blocks instantly. The payoff is the best miss ratio among state-of-the-art policies, and it's running in VMware vSAN today. And all of it — the algorithm and the metadata-trace method — is reproducible in libCacheSim. Thank you, and I'm happy to take questions."

**Remember:** One simple, correlation-aware idea — best miss ratio, and deployed.

---

### Timing summary
| Slide | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| min | 0:30 | 1:00 | 1:15 | 1:15 | 1:00 | 1:30 | 1:30 | 1:00 | 1:15 | 1:00 | 0:45 | 0:45 | **12:00** |

**If you're running long,** trim slide 6 to ~0:45 (let the animation carry the paths) and tighten slide 4 — that recovers ~1:00.
