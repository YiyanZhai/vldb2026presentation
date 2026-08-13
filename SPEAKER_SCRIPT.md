# Clock2Q+ — 12-Minute Speaker Script (VLDB 2026)

Aligned slide-by-slide with `index.html` (13 main slides). Target ≈ **11:30 spoken**, buffer for pacing.
Delivery: ~130 words/min, pause on each figure/fragment reveal. Advance fragments (→) as you reach the matching sentence.

---

## Slide 1 — Title · ~0:30
**Say:**
> "Hi everyone, I'm Yiyan Zhai from Carnegie Mellon — joint work with collaborators at Harvard and Broadcom. I'll talk about **Clock2Q+**, a cache replacement algorithm we built for *metadata* caches, and that's running in production in **VMware vSAN**. The idea is simple, so my goal is that by the end you'll understand exactly *why* it works."

**Remember:** A simple, production-deployed idea — set the expectation that it's intuitive.

---

## Slide 2 — What is a metadata cache? · ~0:55
**Transition:** "Let me start with what we're actually caching."
**Say:**
> "A block cache keeps hot pages in DRAM so you avoid going to SSD or disk. Systems keep two kinds. A **data cache** holds the actual payload — file or disk blocks. A **metadata cache** holds the *index* the system uses to **locate** that data — in vSAN, that's B-trees mapping a logical block number to a physical one. That distinction matters, because the metadata cache is on the **hottest path**: you consult it to find almost every piece of data. So it runs at extreme frequency, under high concurrency, often with hit ratios already near 100%. Which means we judge its policy not only by miss ratio, but by **CPU overhead, scalability, and simplicity**."

**Remember:** Metadata cache = the index (B-trees) that locates data; hottest path → overhead & simplicity are first-class.

---

## Slide 3 — Structural locality: a page packs many keys · ~0:55
**Transition:** "So why does a metadata cache behave differently from a data cache? It starts with structure."
**Say:**
> "A metadata page isn't a single key — a B-tree **leaf packs hundreds of mapping tuples**. So many different keys physically co-reside on the same page. In this example, **L1 and L5 both live in leaf m4**, so two completely unrelated lookups both hit m4. Different accesses land on the same page not because it's popular, but because of how keys are packed — that's **structural locality**. One note: the internal nodes near the root are touched by *every* lookup, so they're effectively pinned; the interesting action is at the leaves, which are 99% of the tree."

**Remember:** A leaf packs hundreds of mapping tuples; distinct keys co-reside → accesses hit the same page by structure, not popularity.

---

## Slide 4 — What is a correlated reference? · ~1:05
**Transition:** "That packing shows up directly in the access stream."
**Say:**
> "Remember, all those keys share a page. So when a run of accesses touches keys that happen to co-reside — say L1, then L5, then L8, all sitting in leaf m4 — that one leaf gets hit again and again in a split second, then goes quiet for a long time. Here's the intuition: picture a single shelf in a library. Different people pull *different* books, but they all come from that *one* shelf — so for a minute the shelf looks incredibly busy. Not because it's special — just because so much lives on it. That's a **correlated reference**: unrelated accesses piling onto the same page because of how it's packed. It's *weak* evidence of long-term value. And the trap every standard policy falls into is reading each of those hits as a separate vote for 'this page is hot.'"

**Remember:** A burst = unrelated accesses piling onto the same page (keys co-reside) — busy by structure, not popularity. Counting each hit as a vote is the trap.

---

## Slide 5 — Why existing policies fall short · ~1:05
**Transition:** "So how do today's best policies handle a burst? They share one basic mechanism — let me define it, then show how each one trips."
**Say:**
> "Both policies I'll compare against work the same basic way: new blocks sit in a small admission queue, and each carries a **reference bit** — just a single yes/no flag recording whether the block has been hit *again* since it arrived. Earn that bit and you're promoted into the main cache; otherwise you're dropped. Now the failure. S3-FIFO — the recent state of the art — flips that bit on *any* re-hit. So a correlated burst flips it on a stone-cold page, and that page gets promoted — pure pollution. The classics, 2Q and Clock2Q, overcorrect: a genuinely hot block can't get promoted until it's been dropped once and *then* asked for again — so every hot block pays one extra miss. Promote on any hit, and you keep cold blocks; demand an extra miss first, and you make the hot blocks wait."

**Remember:** Reference bit = 1/0 "hit again since it arrived?" S3-FIFO sets it on any re-hit → over-promotes bursts; Clock2Q needs a drop-and-re-ask → under-promotes. Want both.

---

## Slide 6 — Key insight: the correlation window · ~0:55
**Transition:** "Here's the insight — and it's almost embarrassingly simple."
**Say:**
> "The realization is that *where* a re-hit happens tells you what it means. If a block is re-hit right after it's inserted — near the head of the small queue — that's almost certainly part of the *same* burst, so ignore it. If it's re-hit later, after it's aged a bit, that's real reuse worth acting on. So we add a **correlation window** at the head of the small queue: hits *inside* the window do **not** set the reference bit; hits *beyond* it do. Inside, bursts are filtered out. Beyond, genuinely hot blocks still get promoted directly. That's it — a one-line change to S3-FIFO, no new tuning knobs."

**Remember:** *Where* the re-hit happens is the signal — one small change to S3-FIFO.

---

## Slide 7 — Clock2Q+ design · ~1:05
**Transition:** "Let me put that window into the full picture."
**Say:**
> "Three queues. New blocks enter the **Small FIFO**, whose head portion is the correlation window. Follow the paths. If a block reaches the tail *without* earning a reference bit — cold, or just a burst inside the window — it ages out to the **Ghost** queue, which keeps only the key. If it earned Ref = 1 *beyond* the window, it's promoted straight into the **Main Clock** — no extra miss. If a key sitting in Ghost is requested again, that's proven reuse, so we admit it directly to Main. And the Main Clock uses the classic second-chance rule: the reference bit buys one more pass of the clock hand before eviction."

**Remember:** Four clean paths — cold/burst → Ghost, hot → Main directly, Ghost-hit → Main, Main uses second-chance.

---

## Slide 8 — Clock2Q+ in action · ~1:15
**Transition:** "Rather than trace that on a static diagram, let's step through it." *(Animation is in step mode — advance with Next ▶ / click / →.)*
**Say (narrate as you step each move):**
> "A **cold** block ages to the tail with its bit still zero, and drops into Ghost. Now a **correlated burst** — it's hit, but *inside the window*, so the bit stays zero; it also ages out to Ghost — the burst never touches Main. Here's a **hot** block — hit *beyond* the window, so it earns Ref = 1, and when it reaches the tail it's promoted straight to Main. Now a **Ghost hit** — a key we remembered comes back, admitted directly to Main. And in the **Main Clock**, a block with the bit set gets a second chance — the hand clears it and moves on, then evicts a cold one. The window does quiet work while the fast path stays fast."

**Remember:** The window silently filters bursts; the direct-promotion fast path is untouched.

---

## Slide 9 — Evaluating a metadata policy · ~0:50
**Transition:** "How do you even evaluate a metadata policy? That's surprisingly hard."
**Say:**
> "There are no public metadata-cache traces, and production traces can't be shared. So here's the trick: take *any* public block trace and divide each block number by the B-tree fan-out — around 200. That reconstructs the sequence of leaf-page accesses a real metadata cache would see. We validated it against a real B-tree trace, and the miss-ratio curves match to within one-hundredth of a percent. So anyone can regenerate metadata results from public data. We ran 106 CloudPhysics traces — over two billion requests — against ten state-of-the-art policies."

**Remember:** A simple, validated recipe makes metadata-cache evaluation reproducible for everyone.

---

## Slide 10 — Miss ratio vs. 10 SOTA policies · ~1:10
**Transition:** "So — does the window actually pay off?"
**Say:**
> "These box plots show relative miss-ratio improvement, with **Clock2Q+ as the zero line** — every other policy is measured against us. On metadata, on the left, every competitor's box sits *below* the line — worse than Clock2Q+. On data traces, on the right, same story. Across every cache size we tested, Clock2Q+ has the best **median and mean** — and on some metadata traces we beat S3-FIFO, the strongest competitor, by up to **28.5%**. The headline isn't one lucky trace — it's that we win *across the board*, on metadata and data."

**Remember:** Best median and mean at every cache size, on metadata *and* data.

---

## Slide 11 — Why it works · ~0:55
**Transition:** "Let me show the mechanism behind that number."
**Say:**
> "This table counts how many blocks each policy promotes from the small queue into the Main Clock. S3-FIFO promotes about 88,000; Clock2Q+ promotes about 21,000 — under a quarter as many. We're far more selective, because the window filters correlated bursts before they can be promoted. And it's not just fewer — it's the *right* ones: the blocks we promote have short reuse distances, so they're genuinely hot, while the ones we ghost have long ones."

**Remember:** We promote under ¼ as many blocks — and they're the genuinely hot ones.

---

## Slide 12 — Engineered for production · ~0:40
**Transition:** "And because this ships in vSAN, miss ratio was never the only requirement."
**Say:**
> "Everything is array-based, no allocation, so the hit path is just a hash lookup and at most flipping one bit. It scales with fine-grained locks. It handles dirty pages by skipping them until they're flushed. It bounds the work per eviction, so no CPU spikes. And it resizes online. All of it came out of nearly a decade of evolving vSAN's cache — with simplicity treated as a hard requirement."

**Remember:** Low overhead, scalable, dirty-aware, bounded, resizable — simplicity by design.

---

## Slide 13 — Takeaways · ~0:40
**Transition:** "So, to wrap up."
**Say:**
> "Correlated references are a real, costly pattern in metadata caches, and they defeat the usual one-hit-wonder filters. The correlation window is a tiny change that fixes it — filter the bursts, still promote truly hot blocks instantly. The payoff is the best miss ratio among state-of-the-art policies, and it's running in VMware vSAN today. And it's all reproducible in libCacheSim. Thank you — happy to take questions."

**Remember:** One simple, correlation-aware idea — best miss ratio, and deployed.

---

### Timing summary
| Slide | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| min | 0:30 | 0:55 | 0:55 | 1:05 | 1:05 | 0:55 | 1:05 | 1:15 | 0:50 | 1:10 | 0:55 | 0:40 | 0:40 | **12:00** |

**If you're running long,** the animation (slide 8) and the design slide (7) are the easiest to compress — let the visuals carry them.
