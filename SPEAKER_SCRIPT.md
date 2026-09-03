# Clock2Q+ — Speaker Script (VLDB 2026)

Aligned slide-by-slide with `index.html` (13 main slides + Thank-you).
Delivery: pause on each figure/fragment reveal. Advance fragments (→) as you reach the matching sentence.

---

## Slide 1 — Title
**Say:**
> "Hi everyone, I'm Yiyan Zhai from Carnegie Mellon — joint work with collaborators at Harvard and Broadcom. I'll talk about **Clock2Q+**, a cache replacement algorithm we built for *metadata* caches, and that's running in production in **VMware vSAN**. The idea is simple, so my goal is that by the end you'll understand exactly *why* it works."

**Remember:** A simple, production-deployed idea — set the expectation that it's intuitive.

---

## Slide 2 — What is a metadata cache?
**Transition:** "Let me start with what we're actually caching."
**Say:**
> "A block cache keeps hot pages in DRAM so you avoid slower SSD or disk. Storage systems keep two types of information. The **data** holds the payload — the actual file blocks. The **metadata** holds the *map* you need to **find** that payload — in vSAN, metadata translate a logical block address to a physical one and they are stored in **B⁺ trees**. And there are caches for both of them. That distinction matters, because the metadata cache is on the **hottest path**: you consult it to find almost every piece of data. So it runs at extreme frequency, under high concurrency, often with hit ratios already near 100%. Which means we judge its policy not only by miss ratio, but by **CPU overhead, scalability, and simplicity**."

**Remember:** Data = payload; metadata = the map (B⁺ trees in vSAN) that finds it — both are cached; hottest path → overhead & simplicity are first-class.

---

## Slide 3 — Structural locality: a page packs many keys
**Transition:** "So why does a metadata cache behave differently from a data cache? It starts with structure."
**Say:**
> "One sentence on the structure, since everything follows from it. A **B⁺ tree** is the standard *ordered* index — the one every database uses for range lookups. The internal nodes are just signposts: they **route** a lookup down the tree. **All** the actual key-to-location tuples live in the **leaf** pages at the bottom, and leaves are **99% or more** of the pages. So the internal nodes near the root are touched by *every* lookup and are effectively pinned — the interesting action is at the leaves.
>
> Now the key fact: **one leaf page packs hundreds of those mapping tuples**. So many different keys physically co-reside on the same page. In this figure, **L1 and L5 both live in Leaf m4** — different keys, one page. Different accesses hit the same page not because it's popular, but because of how keys are packed — that's **structural locality**. And it isn't a B⁺-tree quirk: extent maps, bitmaps, inode tables all pack many entries into one page, so they behave the same way."

**Remember:** B⁺ tree = ordered index; internal nodes route, all tuples live in the leaves (99%+ of pages). One leaf packs hundreds of tuples → co-resident keys (L1 & L5 → m4) hit the same page by structure, not popularity. Not B⁺-tree-specific.

---

## Slide 4 — What is a correlated reference?
**Transition:** "That packing shows up directly in the access stream."
**Say:**
> "When a workload scans a run of nearby keys — say L1, then L2, then L3 — and they all live in the same leaf, **Leaf m4**, that one leaf gets hit several times in a split second, then goes cold for a long while. Here's the intuition: picture scanning one shelf in a library — you pull several *neighboring* books in a row, so for a moment that shelf looks incredibly busy. Not because it's special — just because your scan happened to pass through it. That's a **correlated reference**: **sequential lookups to a cold page, bunched into a short time window**. It's *weak* evidence of long-term value. And the trap every standard policy falls into is reading each of those hits as a separate vote for 'this page is hot.'"

**Remember:** A correlated reference = sequential lookups to a *cold* page in a short time (co-resident keys), not unrelated random hits. Busy by structure, not popularity — counting each hit as a vote is the trap.

---

## Slide 5 — Three-queue skeleton (shared design)   ← NEW
**Transition:** "Before we compare the policies, here's the skeleton they all share."
**Build (→, six clicks):** Small · Main · Ghost (arriving with the *not promoted → evict data, remember key* drop that fills it) — then path **①** direct promotion and path **②** re-request (a block travels each arrow) — then the closing callout.
**Say:**
> "2Q, S3-FIFO, and Clock2Q are all built from the same **three queues**. A new block first enters a **Small FIFO** — think of it as **probation**: it has to prove reuse before it can occupy the protected cache. The **Main** queue is **protection** — the long-lived blocks judged worth keeping. And the **Ghost** is **history** — it stores *keys only*, no data, remembering what was recently evicted. A block reaches Main one of two ways.
>
> Path **one** — **qualifying reuse** while it's still on probation in the Small FIFO: it's hit again before it ages out, so it's promoted **directly** into Main.
>
> Path **two** — a **re-request after eviction**, coming back through the Ghost. Say the block reaches the tail of the Small FIFO without qualifying. Its **data is dropped** — that space goes back to the cache — but its **key is kept** in the Ghost. So the cache has forgotten the *contents* and remembered only that *it saw this block recently*. If that key is then requested again, we take the miss and fetch from storage — but the Ghost hit tells us this isn't a first-time block: it's been asked for twice, separated by a whole pass through the Small FIFO. That's exactly the evidence probation was there to collect, so the block is admitted **straight into Main** instead of starting over on probation. And because the Ghost stores keys only — no data — you can remember far more history than you could ever cache, for almost nothing.
>
> So: promoted from probation, or promoted on the way back through history. The architecture is fixed; what differs between the policies is exactly **how an item earns promotion to Main** — and, as we'll see, the two paths trade off against each other."

**Remember:** Three queues — Small (probation), Main (protection), Ghost (history, keys only). Two paths to Main: (1) qualifying reuse in Small → direct promotion; (2) re-request after eviction → Ghost keeps the key when the data is dropped, so a later hit on that key proves a *second, separated* access and admits the block straight to Main (one miss paid). Keys-only = cheap, long history. Policies differ only in what counts as "qualifying reuse."

---

## Slide 6 — Why existing policies fall short
**Transition:** "So, given that shared skeleton — the two policies we compare answer 'what qualifies for Main?' in opposite, and both wrong, ways."
**Say:**
> "S3-FIFO — the recent state of the art — uses a **reference bit**: a single yes/no flag, 'has this block been hit *again* since it arrived?' *Any* re-hit flips the bit, and the block is promoted to Main. So a correlated burst flips it on a cold page — promoted, pure pollution. The classics, 2Q and Clock2Q, have *no* such bit: a hot block can't be promoted until it's been **evicted to the Ghost and then requested again** — so every hot block pays one extra miss. Now you see the two opposite mistakes: promote on any hit, and you keep cold blocks; demand an extra miss first, and you evict the hot ones before they earn their place."

**Remember:** S3-FIFO's ref bit — 1/0 "hit again?", set on any re-hit → over-promotes bursts. Clock2Q/2Q have no bit → evict-and-re-ask → under-promote (extra miss). Want both fixed.

---

## Slide 7 — Clock2Q+: add a correlation window
**Transition:** "So — promote on any hit and you keep cold blocks; wait too long and hot blocks miss. Clock2Q+ changes exactly one thing, and it comes down to *where* a re-hit happens."
**Say:**
> "In **S3-FIFO**, *any* re-hit on a block still in the Small FIFO sets Ref = 1 — so a correlated burst promotes a cold page. Clock2Q+ keeps the same three queues and adds one thing: a **correlation window** at the head of the Small FIFO that **filters the correlated burst** — a re-hit *inside* the window does **not** set the bit. *(Click the diagram to toggle the window on and off.)*
>
> The realization is that *where* a re-hit happens tells you whether the popularity is real. If a block is re-hit right after it's inserted — inside the window — that's part of the *same* burst, so we ignore it. If it's re-hit later, once it's aged past the window, that's genuine reuse worth acting on. *(The two buttons demo exactly that — hit inside vs. beyond.)*
>
> So: inside the window, bursts are filtered out; beyond it, truly hot blocks still promote directly to Main, no extra miss. That's the entire contribution — one rule on top of S3-FIFO, no new knobs."

**Remember:** One bit tells the whole story — S3-FIFO sets Ref = 1 on any hit (burst → pollution); Clock2Q+ keeps in-window hits at 0. Inside → Ghost, beyond → Main. That window is the only change.

---

## Slide 8 — Clock2Q+ in action
**Transition:** "Let's watch it — I'll step through one move at a time." *(Step mode: Next ▶ / click / →.)*
**Say (narrate as you step each move):**
> "A **cold** block ages to the tail with its bit still zero, and drops into Ghost. A **correlated burst** is hit *inside* the window, so its bit stays zero and it too ages out to Ghost. A **hot** block hit *beyond* the window earns Ref = 1 and, at the tail, is promoted straight to Main — entering the Main Clock fresh, with its bit reset to zero. A **ghost hit** — a remembered key — is admitted directly to Main. And in the Main Clock, a block with its bit set gets a second chance before a cold one is evicted; the hand only ever sweeps clockwise. The **Correlated burst** and **Hot block** buttons jump to those cases as a snapshot of the *same* run — same object queue, not a fresh example. **So that's the whole policy in motion: the window quietly filters the correlated bursts, while genuinely hot blocks still promote directly.** From here, everything answers one question: does it actually help?"

**Remember:** Close by naming it — window filters bursts, hot blocks promote directly (entering Main with Ref=0). Scenario buttons are snapshots of the same run. Then pivot to "does it help?"

---

## Slide 9 — Method: generating metadata traces
**Transition:** "How do you even evaluate a metadata policy? That's surprisingly hard."
**Say:**
> "There are no public metadata-cache traces, and production traces can't be shared. So here's the trick: take *any* public block trace and divide each block number by the B⁺-tree fan-out — around 200. That reconstructs the sequence of leaf-page accesses a real metadata cache would see. We validated it against a real B⁺-tree trace, and the miss-ratio curves match to within one-hundredth of a percent. So anyone can regenerate metadata results from public data. We ran 106 CloudPhysics traces — over two billion requests — against ten state-of-the-art policies."

**Remember:** A simple, validated recipe (block# ÷ fan-out) makes metadata-cache evaluation reproducible for everyone.

---

## Slide 10 — Lowest miss ratio, on metadata and data
**Transition:** "So — does the window actually pay off?"
**Say:**
> "These box plots show relative miss-ratio improvement, with **Clock2Q+ as the zero line** — every other policy is measured against us. On metadata, on the left, every competitor's box sits *below* the line — worse than Clock2Q+. On data traces, on the right, same story. Across every cache size we tested, Clock2Q+ has the best **median and mean** — and on some metadata traces we beat S3-FIFO, the strongest competitor, by up to **28.5%**. And independently: in Juncheng Yang's course at Harvard, students implement a range of cache-replacement policies — Clock2Q+ comes out best. The headline isn't one lucky trace — we win *across the board*, on metadata and data."

**Remember:** Best median and mean at every cache size, on metadata *and* data. (28.5% is per-trace, say it second.) Harvard-course result = independent validation.

---

## Slide 11 — Where Clock2Q+ comes from (design lineage)
**Transition:** "And Clock2Q+ didn't appear from nowhere — here's its design lineage."
**Say:**
> "**2Q** is the conceptual starting point — a Small FIFO, a Main LRU, and a Ghost. From it, two branches fix different things. **Clock2Q** swaps the LRU for a **Clock** — the same idea at far lower CPU — and that's what actually shipped: single-threaded around 2010 in vSAN OSA, then a concurrent version in VDFS and early ESA. Separately, **S3-FIFO** adds a **reference bit** to the Small FIFO, so re-referenced blocks promote directly — but, as we saw, correlated bursts can look hot. **Clock2Q+** brings both lines together: from Clock2Q it takes the low-overhead Clock and adds the Ref bit; from S3-FIFO it takes direct promotion and adds the **correlation window**. The result is correlation-aware, and it's what runs in vSAN — VDFS, ESA, and more — today. Through all of it, simplicity stayed a hard requirement."

**Remember:** 2Q (concept) → two branches: Clock2Q (LRU→Clock; OSA ~2010 single-thread, then concurrent VDFS/ESA) and S3-FIFO (Ref bit, over-promotes) → converge on Clock2Q+ (+ Ref bit + correlation window). Simplicity throughout.

---

## Slide 12 — Built to run safely at scale
**Transition:** "So how does it hold up as real, shipping code?"
**Say:**
> "Because it ships, miss ratio was never the only bar. The real constraint is **concurrency**. Under many CPUs, **LRU is a liability** — it reorders on *every* hit, which means a write-lock per access and contention at scale; that's exactly why we use a **Clock** — its reference bit is read-mostly, so it scales on fine-grained locks with no global lock. The hit path is a hash lookup and at most one bit flip, all array-based, no allocation. It skips dirty pages until they're flushed, bounds the work per eviction — no CPU spikes, no infinite loop even if every page is dirty — and it resizes online. The bigger picture: textbooks teach LRU, but at multi-CPU scale Clock is what actually runs safely. Simplicity was the constraint the whole way through."

**Remember:** Concurrency is the real constraint — LRU reorders per hit → lock contention; Clock (read-mostly Ref bit) scales. Also dirty-aware, bounded, resizable. Textbooks say LRU; at scale, Clock.

---

## Slide 13 — Takeaways
**Transition:** "So, to wrap up."
**Say:**
> "Correlated references are a real, costly pattern in metadata caches, and they defeat the usual one-hit-wonder filters. The correlation window is a tiny change that fixes it — filter the bursts, still promote truly hot blocks instantly. The payoff is the best miss ratio among state-of-the-art policies, and it's running in VMware vSAN today. And it's all reproducible in libCacheSim."

**Remember:** One simple, correlation-aware idea — best miss ratio, and deployed. Reproducible in libCacheSim.

*(Final slide is Thank-you — contacts, related sites, and the libCacheSim QR. "Thank you — happy to take questions.")*

---

**If you're running long,** the new three-queue slide (5) and the animation (8)/design (7) are easiest to compress — let the visuals carry them; slide 5 can be trimmed to just naming the three queues and the two promotion paths.

---

## Backup — Why it works · (on demand)
**Say (if asked why it wins):**
> "This table counts how many blocks each policy promotes from the small queue into the Main Clock. S3-FIFO promotes about 88,000; Clock2Q+ about 21,000 — under a quarter as many. We're far more selective, because the window filters correlated bursts before they can be promoted. And it's the *right* ones: promoted blocks have short reuse distances (genuinely hot); ghosted ones have long distances (cold)."

**Remember:** Under ¼ as many promotions — and the genuinely hot ones.
