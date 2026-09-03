# Clock2Q+ — Stage Cue Cards

One line to get in, one line to land. Full text in `SPEAKER_SCRIPT.md`; live version toggles with **S** in the deck.

| # | ↪ Transition in | ★ Remember (land this) |
|---|-----------------|------------------------|
| **1** Title | Name + one-line; simple & deployed. | Simple, production idea — promise intuition. |
| **2** Metadata cache | "What we're actually caching." *(5 clicks: tiers · text + pair on disk · lift into DRAM · hot path · callout)* | Data = payload; **metadata = the map (B⁺ trees in vSAN) that finds it** — both are cached. Hottest path → overhead & simplicity matter too. |
| **3** Structural locality | "Why is metadata different? Start with structure." | **B⁺ tree** = ordered index; internal nodes route, all tuples live in the **leaves** (99%+ of pages). One leaf packs **hundreds of tuples**; L1 & L5 → Leaf m4. Same page by **structure, not popularity**. Not B⁺-tree-specific. |
| **4** Correlated ref | "That packing shows up in the access stream." | **Sequential lookups to a cold page in a short time** (co-resident keys L1,L2,L3 → m4) — not unrelated random hits. Each hit looks like a vote (the trap). |
| **5** Three-queue skeleton | "The skeleton all three policies share." *(6 clicks: Small · Main · Ghost+drop · ① · ② · callout)* | **Small = probation · Main = protection · Ghost = history (keys only).** Two paths to Main: **qualifying reuse** (direct) or **re-request** via Ghost — data dropped, **key kept**, so a later hit = a *second, separated* access → admit straight to Main (one miss paid). Policies differ only in what qualifies. |
| **6** Why others fail | "Two opposite mistakes on 'what qualifies for Main?'" | **Ref bit is S3-FIFO's** (any re-hit → admits a cold page). Clock2Q/2Q: no bit → **evict to Ghost + re-ask** → hot blocks pay an extra miss. |
| **7** Design (+ window) | "The fix — it's about *where* a re-hit happens." | Add a **correlation window**: in-window hits don't set Ref → Ghost; beyond → Ref=1 → Main. The only change vs. S3-FIFO. |
| **8** Animation | "Let's step through it." *(Next ▶ / click)* | Window filters bursts; hot blocks promote directly (enter Main with **Ref=0**); hand sweeps clockwise. Scenario buttons = snapshot of the *same* run. |
| **9** Trace method | "How do you even evaluate a metadata policy?" | Divide block# by fan-out → metadata trace; validated <0.01%. Reproducible. |
| **10** Results | "Does the window actually pay off?" | Best median & mean at every cache size — metadata **and** data. *(28.5% second, per-trace.)* **Harvard course: Clock2Q+ wins** (independent). |
| **11** Design lineage | "Where Clock2Q+ comes from." | 2Q (concept) → **Clock2Q** (LRU→Clock; OSA ~2010, then concurrent VDFS/ESA) **& S3-FIFO** (Ref bit) → converge on **Clock2Q+** (+ Ref bit + correlation window). Simplicity throughout. |
| **12** Run safely at scale | "How does it hold up as shipping code?" | **Concurrency is the real constraint** — LRU reorders per hit → lock contention; **Clock scales**. Dirty-aware · bounded · resizable. Textbooks say LRU; at scale, Clock. |
| **13** Takeaways | "To wrap up." | One correlation-aware idea → best miss ratio + deployed. Reproducible in libCacheSim. |

("Why it works" is a backup.) Running long → compress the three-queue slide (5) and let the animation (8)/design (7) visuals carry them.
