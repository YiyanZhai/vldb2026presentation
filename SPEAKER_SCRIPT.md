# Clock2Q+ — 12-Minute Speaker Script (VLDB 2026)

Aligned slide-by-slide with `index.html` (12 main slides). Target ≈ **11:30 spoken**, buffer for pacing.
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
> "Both policies we compare put new blocks in a small admission queue, and the question is the same: which block in the queue is valuable and should go to the main cache? S3-FIFO — the recent state of the art — uses a **reference bit**: a single yes/no flag, 'has this block been hit *again* since it arrived?' Any re-hit flips the bit, and the block is promoted to main cache. So a correlated burst flips it on a cold page — promoted, pure pollution. The classics, 2Q and Clock2Q, have *no* such bit: a hot block can't be promoted until it's been dropped to a ghost list and *then* requested again — so every hot block pays one extra miss. Now you see the problem: promote on any hit, and you keep cold blocks; demand an extra miss first, and you make the hot blocks wait."

**Remember:** The reference bit is S3-FIFO's — 1/0 "hit again?", set on any re-hit → over-promotes bursts. Clock2Q/2Q have no such bit → need a drop-and-re-ask → under-promote. Want both.

---

## Slide 6 — Clock2Q+: add a correlation window · ~1:30
**Transition:** "Here's the fix — and it's almost embarrassingly simple. It comes down to *where* a re-hit happens."
**Say:**
> "Clock2Q+ keeps exactly S3-FIFO's three queues — a Small FIFO, a Main Clock, and a Ghost — and adds one thing: a **correlation window** at the head of the Small FIFO, where a re-hit does **not** set the reference bit. *(Click the diagram to toggle the window on and off.)* Here's the whole comparison in a single bit. In **S3-FIFO** — window off — *any* re-hit sets Ref = 1, even a burst right at the head, so a cold page gets promoted and pollutes the cache. Turn the window on — that's **Clock2Q+** — and the same in-window hit stays at **0**, so the burst just ages out to Ghost. A re-hit *beyond* the window is genuine reuse, so it does set Ref = 1 and goes straight to Main, no extra miss. *(The two buttons demo exactly that — hit inside vs. beyond.)* That's the entire contribution: one rule on top of S3-FIFO, no new knobs."

**Remember:** One bit tells the whole story — S3-FIFO sets Ref = 1 on any hit (burst → pollution); Clock2Q+ keeps in-window hits at 0. Inside → Ghost, beyond → Main. That window is the only change.

---

## Slide 7 — Clock2Q+ in action · ~1:15
**Transition:** "Let's watch it — I'll step through one move at a time." *(Step mode: Next ▶ / click / →.)*
**Say (narrate as you step each move):**
> "A **cold** block ages to the tail with its bit still zero, and drops into Ghost. A **correlated burst** is hit *inside* the window, so its bit stays zero and it too ages out to Ghost. A **hot** block hit *beyond* the window earns Ref = 1 and, at the tail, is promoted straight to Main. A **ghost hit** — a remembered key — is admitted directly to Main. And in the Main Clock, a block with its bit set gets a second chance before a cold one is evicted. I can also jump straight to the correlated-burst or hot-block case with the buttons. **So that's the whole policy in motion: the window quietly filters the correlated bursts, while genuinely hot blocks still promote directly — no extra miss.** From here, everything answers one question: does it actually help?"

**Remember:** Close by naming it — that's the whole policy: window filters bursts, hot blocks promote directly. Then pivot to "does it help?"

---

## Slide 8 — Evaluating a metadata policy · ~0:50
**Transition:** "How do you even evaluate a metadata policy? That's surprisingly hard."
**Say:**
> "There are no public metadata-cache traces, and production traces can't be shared. So here's the trick: take *any* public block trace and divide each block number by the B-tree fan-out — around 200. That reconstructs the sequence of leaf-page accesses a real metadata cache would see. We validated it against a real B-tree trace, and the miss-ratio curves match to within one-hundredth of a percent. So anyone can regenerate metadata results from public data. We ran 106 CloudPhysics traces — over two billion requests — against ten state-of-the-art policies."

**Remember:** A simple, validated recipe makes metadata-cache evaluation reproducible for everyone.

---

## Slide 9 — Miss ratio vs. 10 SOTA policies · ~1:10
**Transition:** "So — does the window actually pay off?"
**Say:**
> "These box plots show relative miss-ratio improvement, with **Clock2Q+ as the zero line** — every other policy is measured against us. On metadata, on the left, every competitor's box sits *below* the line — worse than Clock2Q+. On data traces, on the right, same story. Across every cache size we tested, Clock2Q+ has the best **median and mean** — and on some metadata traces we beat S3-FIFO, the strongest competitor, by up to **28.5%**. The headline isn't one lucky trace — it's that we win *across the board*, on metadata and data."

**Remember:** Best median and mean at every cache size, on metadata *and* data.

---

## Slide 10 — How vSAN's cache evolved · ~0:45
**Transition:** "And Clock2Q+ didn't appear from nowhere."
**Say:**
> "It's the latest step in nearly a decade of evolving vSAN's cache. It began with **2Q** — a small FIFO ahead of a main LRU. **Clock2Q** swapped that LRU for a **Main Clock** — the same idea at far lower CPU — and shipped across vSAN OSA, VDFS, and early ESA. Then **S3-FIFO** added the **reference bit**, so hot blocks promote directly — but, as we saw, it over-promotes correlated bursts. **Clock2Q+** is Clock2Q plus that bit plus the correlation window that filters the bursts — and it's what runs in vSAN and vSAN File Services today. Through all of it, simplicity stayed a hard requirement."

**Remember:** 2Q → Clock2Q (Main Clock) → S3-FIFO (Ref bit) → Clock2Q+ (+ window). ~A decade in vSAN; simplicity throughout.

---

## Slide 11 — Engineered for production · ~0:45
**Transition:** "So how does it hold up as real, shipping code?"
**Say:**
> "Because it ships, miss ratio was never the only bar. The hit path is a hash lookup and at most one bit flip; everything is array-based, no allocation. It scales on fine-grained locks — a mutex per hash bucket, atomic queue indices, no global lock. It skips dirty pages until they're flushed, and it bounds the work per eviction — so no CPU spikes, and no infinite loop even if every page is dirty. And it resizes online. Simplicity was the constraint the whole way through."

**Remember:** Low overhead, scalable, dirty-aware, bounded, resizable — simplicity by design.

---

## Slide 12 — Takeaways · ~0:40
**Transition:** "So, to wrap up."
**Say:**
> "Correlated references are a real, costly pattern in metadata caches, and they defeat the usual one-hit-wonder filters. The correlation window is a tiny change that fixes it — filter the bursts, still promote truly hot blocks instantly. The payoff is the best miss ratio among state-of-the-art policies, and it's running in VMware vSAN today. And it's all reproducible in libCacheSim. Thank you — happy to take questions."

**Remember:** One simple, correlation-aware idea — best miss ratio, and deployed.

---

### Timing summary
| Slide | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| min | 0:30 | 0:55 | 0:55 | 1:05 | 1:05 | 1:30 | 1:15 | 0:50 | 1:10 | 0:45 | 0:45 | 0:40 | **11:25** |

**If you're running long,** the animation (slide 7) and the design slide (6) are the easiest to compress — let the visuals carry them.


---

## Backup — Why it works · (on demand)
**Say (if asked why it wins):**
> "This table counts how many blocks each policy promotes from the small queue into the Main Clock. S3-FIFO promotes about 88,000; Clock2Q+ about 21,000 — under a quarter as many. We're far more selective, because the window filters correlated bursts before they can be promoted. And it's the *right* ones: promoted blocks have short reuse distances (genuinely hot); ghosted ones have long distances (cold)."

**Remember:** Under ¼ as many promotions — and the genuinely hot ones.
