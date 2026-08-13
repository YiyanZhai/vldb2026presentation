# Clock2Q+ — 12-Minute Speaker Script (VLDB 2026)

Aligned slide-by-slide with `index.html` (13 main slides). Target ≈ **11:30 spoken**, with buffer.
Delivery: ~130 wpm, unhurried. `[click]` = reveal next fragment; `[step]` = advance the animation; `[pause]` = beat for effect. Say the **bold** words with weight.

---

## Slide 1 — Title · ~0:30  *(the hook)*
**Say:**
> "Here's a claim I'd like to convince you of in the next twelve minutes: one of our biggest wins for a *production* cache wasn't a clever new algorithm — it was deciding, in one specific spot, to **ignore a cache hit**. [pause] I'm Yiyan Zhai from Carnegie Mellon; joint work with Harvard and Broadcom. The system is **Clock2Q+**, the metadata-cache policy running today in **VMware vSAN**."

**★ Remember:** Open on the paradox — *sometimes you should ignore a hit.* Promise it's simple.

---

## Slide 2 — What is a metadata cache? · ~0:55
**Transition:** "First — what are we caching, and why is one cache special?"
**Say:**
> "A block cache keeps hot pages in DRAM so you don't pay for SSD or disk. Storage systems keep *two* of them. The **data cache** holds the payload — the actual file or disk blocks. The **metadata cache** holds the map you need to *find* that payload — in vSAN, B-trees that translate a logical block number to a physical one. [click] And that second cache is special, because you touch it to locate almost **every** piece of data. It runs constantly, under heavy concurrency, usually when the hit ratio is already near 100%. So here a good policy is judged on more than miss ratio — CPU cost, scalability, and plain **simplicity** are just as real."

**★ Remember:** Metadata cache = the *map that finds data*; it's the hot path → overhead & simplicity are first-class.

---

## Slide 3 — Structural locality: a page packs many keys · ~0:55
**Transition:** "So why does that map behave so differently from a data cache? It comes down to how it's laid out."
**Say:**
> "A metadata page isn't one key — a B-tree leaf packs *hundreds* of entries; in vSAN, about **200**. So totally unrelated keys end up living on the same physical page. [click] Here, **L1 and L5 both sit in leaf m4** — so two unrelated lookups both land on m4, back to back. That page looks busy — but not because anyone loves it, purely because of how the keys are packed. We call that **structural locality**. [click] One aside: the nodes near the root get hit on *every* lookup, so they're effectively pinned — the interesting behavior is all at the leaves, which are 99% of the tree."

**★ Remember:** A leaf packs ~200 keys → unrelated keys co-reside; pages look busy by *structure*, not popularity.

---

## Slide 4 — What is a correlated reference? · ~1:05
**Transition:** "Line those co-resident accesses up in time, and you get the pattern at the heart of this talk."
**Say:**
> "Because a page holds so many keys, a single operation — traversing or updating an index — touches a bunch of them on the **same** page. So that page gets slammed a handful of times in a split second… and then goes quiet, maybe for a very long time. [pause] Think of grabbing five books off *one* shelf on a single trip to the library: for one minute that shelf looks wildly popular — but you won't be back for weeks. That's a **correlated reference** — many accesses collapsing onto one page, bunched in time. It's *weak* evidence of long-term value. And here's the trap: every standard policy counts each of those hits as a separate vote for 'keep me.'"

**★ Remember:** A burst is one errand, not a hot page — counting each hit as a vote is *the trap*.

---

## Slide 5 — Why existing policies fall short · ~1:05
**Transition:** "Now watch what that trap does to the two families of policies you'd reach for."
**Say:**
> "S3-FIFO — today's state of the art — puts new blocks in a small queue with a reference bit, and any re-hit flips the bit and promotes the block. Under a correlated burst, a stone-cold page gets a few hits near the head, flips its bit, and lands in the main cache — pure **pollution**. [click] The classics, 2Q and Clock2Q, overcorrect: a genuinely hot block can't reach main until it's been evicted *and* asked for again — so every hot block eats one **extra miss**. So you're pinned between two bad options: too eager, you swallow the bursts; too cautious, you punish the real hits. We want direct promotion — *without* the bursts."

**★ Remember:** S3-FIFO over-promotes; Clock2Q under-promotes. We want direct promotion without the bursts.

---

## Slide 6 — Key insight: the correlation window · ~0:55
**Transition:** "The fix turned out to be almost silly in hindsight."
**Say:**
> "The whole trick is realizing that **where** a re-hit happens tells you what it means. A re-hit right after insertion — near the head of the small queue — is almost certainly the *same burst*, so it's noise: ignore it. A re-hit *later*, once the block has aged a bit, is real reuse: act on it. [click] So we carve out a **correlation window** at the head of the queue where hits simply don't set the reference bit. Inside the window, bursts are filtered for free. Outside it, truly hot blocks still shoot straight to the main cache. No new knob, no sorted list — **one rule**."

**★ Remember:** *Where* the re-hit happens is the signal — ignore hits inside the window. One rule.

---

## Slide 7 — Clock2Q+ design · ~1:05
**Transition:** "Here's that one rule inside the full structure."
**Say:**
> "Three queues. New blocks land in the **Small FIFO**, and its head is the correlation window. Follow a block to the tail. If it never earned a reference bit — cold, or just a burst inside the window — it ages out to the **Ghost** queue, which remembers only the *key*, not the data. [click] If it *did* earn the bit beyond the window, it's promoted straight into the **Main Clock** — no detour. If a key we ghosted comes back, that's proof it's hot, so we admit it to Main directly. [click] And Main is a clock: the reference bit buys a block one more sweep of the hand before it's evicted. That's the whole machine."

**★ Remember:** cold/burst → Ghost · hot → Main directly · Ghost-hit → Main · Main = second-chance clock.

---

## Slide 8 — Clock2Q+ in action · ~1:15
**Transition:** "Let me actually run it — I'll step through one move at a time." *(Step mode: Next ▶ / click / →.)*
**Say (narrate as you step):**
> "Cold block: it drifts to the tail, bit still zero, and drops into Ghost. [step] Now a correlated burst — it's hit, but **inside** the window, so the bit stays zero, and it ages out to Ghost too. Notice — the burst never even touches Main. [step] A hot block — hit **beyond** the window, so it earns its bit, and at the tail it jumps straight into Main. [step] A ghost hit — a key we remembered comes back and goes right into Main. [step] And eviction: a block with its bit set gets a second chance — the hand clears it and moves on — then a cold one goes. The window is doing quiet work the whole time, while the fast path stays fast."

**★ Remember:** The window quietly filters bursts; the fast promotion path is untouched.

---

## Slide 9 — Evaluating a metadata policy · ~0:50
**Transition:** "Quick but important — how do you even measure this honestly?"
**Say:**
> "There are no public metadata-cache traces, and we can't ship customer ones. So here's the move: take *any* public block trace and divide each block number by the B-tree fan-out — about 200. That reconstructs the leaf-page accesses a metadata cache would actually see. [click] We checked it against a real B-tree trace — the miss-ratio curves agree to within a hundredth of a percent. So anyone can reproduce our metadata results from public data. On that basis we ran **106 CloudPhysics traces** against ten state-of-the-art policies."

**★ Remember:** Divide block# by fan-out → a *validated, reproducible* metadata trace anyone can regenerate.

---

## Slide 10 — Miss ratio vs. 10 SOTA policies · ~1:10
**Transition:** "So — does ignoring those hits actually pay off? Here's every policy at once."
**Say:**
> "These box plots put **Clock2Q+ at the zero line** and measure everyone else against it. Metadata on the left: every competitor's box sits *below* zero — worse than us. Data on the right: same picture. [click] Across every cache size we tested, Clock2Q+ has the best median **and** mean, and on some metadata traces we beat S3-FIFO — the strongest baseline — by up to **28.5%**. But the real headline isn't that one number; it's that there's **no regime where anyone else wins**. We're at or above the whole field, everywhere — metadata and data."

**★ Remember:** Best median & mean at every size, metadata *and* data — nobody wins in any regime.

---

## Slide 11 — Why it works · ~0:55
**Transition:** "And it's worth seeing *why*, because it's a one-number story."
**Say:**
> "This counts how many blocks each policy pushes from the small queue into Main. S3-FIFO pushes about **88,000**; we push about **21,000** — under a quarter. That gap *is* the correlated bursts, filtered instead of promoted. [click] And it's not just fewer — it's better: the blocks we promote have short reuse distances, so they really are hot; the ones we ghost have long ones. Fewer promotions, and the *right* promotions — the whole win in one table."

**★ Remember:** Under ¼ as many promotions — and they're the genuinely hot ones.

---

## Slide 12 — Engineered for production · ~0:40
**Transition:** "One last thing — this isn't a paper prototype, it ships."
**Say:**
> "The hit path is a hash lookup and maybe one bit flip; everything's array-based, no allocation. It scales on fine-grained locks, skips dirty pages until they flush, caps the work per eviction so there are no CPU spikes, and resizes online. All of it came out of nearly a decade of evolving vSAN's cache — where **simplicity wasn't a preference, it was a requirement**."

**★ Remember:** Low overhead, scalable, dirty-aware, bounded, resizable — simplicity was required.

---

## Slide 13 — Takeaways · ~0:40  *(callback)*
**Transition:** "So — back to my opening promise."
**Say:**
> "The one idea to take home: in a metadata cache, a hit **isn't always a vote** to keep a page — correlated bursts fake popularity. Clock2Q+ fixes that by *ignoring* hits in a small window at the head of the queue — a one-line change. The payoff is the best miss ratio among state-of-the-art policies, and it's running in vSAN today, all reproducible in libCacheSim. Thank you — I'd love your questions."

**★ Remember:** Sometimes ignoring a hit is the win — one line, best miss ratio, deployed.

---

### Timing summary
| Slide | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| min | 0:30 | 0:55 | 0:55 | 1:05 | 1:05 | 0:55 | 1:05 | 1:15 | 0:50 | 1:10 | 0:55 | 0:40 | 0:40 | **12:00** |

**If running long:** let the animation (8) and design (7) visuals carry the words; trim slide 5's second half.

---

## Q&A — likely questions (crisp answers)

- **"What exactly is a metadata cache vs. a data cache?"** Data cache = the payload blocks. Metadata cache = the index that *locates* the payload — in vSAN, B-trees mapping LBN→PBN. It's consulted for nearly every data access, so it's the hotter path.
- **"Why do unrelated accesses hit the same page?"** Structural locality: a B-tree leaf packs ~200 tuples, so distinct keys co-reside (L1 & L5 → leaf m4). vSAN actually uses *two* B-trees (Logical: LBN→MBA, Middle: MBA→PBN), which compounds it.
- **"Aren't internal/root nodes constantly hit?"** Yes — they're effectively pinned by any reasonable policy, so we focus on leaf pages, which are 99%+ of the tree and where eviction decisions actually matter.
- **"Couldn't S3-FIFO fix this by tuning, or a bigger ghost?"** No — its reference bit can't tell a correlated burst from real reuse; both flip it. The window adds exactly that missing distinction. We also *shrank* the ghost (to 50%) and still win.
- **"How do you set the window size?"** Default 50% of the Small FIFO. It's insensitive: 10% / 30% / 50% give essentially the same miss ratio (sensitivity backup) — so it's not a tuning knob in practice.
- **"Is 'divide by fan-out' a real workload?"** It's validated: derived metadata traces match a real B-tree leaf trace to <0.01% miss ratio. It's a reproducibility tool, not the deployment.
- **"Where does it *not* help?"** Object/KV workloads with few correlated references (Wikimedia, Meta KV, Tencent): there Clock2Q+ is on par with, and slightly behind, S3-FIFO — honest limitation, in backup.
- **"Throughput / scalability numbers?"** Hit path is O(1) (hash lookup + optional bit); synchronization is per-entry / per-bucket, and cache-lock contention didn't show up as a bottleneck in vSAN/VDFS testing. (No standalone throughput benchmark in the paper.)
- **"Does the dirty-page simplification hurt accuracy?"** Negligible in simulation — at the smallest cache size it can even help; slightly worse for a few traces at large sizes (dirty backup).
- **"Is 28.5% the average?"** No — it's the *largest per-trace* gap over S3-FIFO-2bit at the larger cache size. The defensible headline is best *median and mean* at every size.
