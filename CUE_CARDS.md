# Clock2Q+ — Stage Cue Cards

One line to get in, one line to land. Full text in `SPEAKER_SCRIPT.md`; live version toggles with **S** in the deck.

| # | Time | ↪ Transition in | ★ Remember (land this) |
|---|------|-----------------|------------------------|
| **1** Title | 0:30 | Name + one-line; simple & deployed. | Simple, production idea — promise intuition. |
| **2** Metadata cache | 0:55 | "What we're actually caching." | Data cache = payload; **metadata cache = the index (B-trees) that locates data**. Hottest path → overhead & simplicity matter too. |
| **3** Structural locality | 0:55 | "Why is metadata different? Start with structure." | A leaf packs **hundreds of mapping tuples**; L1 & L5 → leaf m4. Same page by **structure, not popularity**. |
| **4** Correlated ref | 1:05 | "That packing shows up in the access stream." | Burst = **unrelated accesses on the same page** (keys co-reside; L1,L5,L8 → m4). Busy by structure, not popularity — each hit looks like a vote (the trap). |
| **5** Why others fail | 1:05 | "Both share one mechanism — define it first." | **Ref bit is S3-FIFO's** (1/0 "hit again?") — any re-hit sets it → admits a cold page. Clock2Q/2Q: no bit → only after drop + re-ask → delays hot blocks (extra miss). |
| **6** Design (+ window) | 1:30 | "The fix — it's about *where* a re-hit happens." | Add a **correlation window**: in-window hits don't set Ref → Ghost; beyond → Ref=1 → Main. The only change vs. S3-FIFO. |
| **7** Animation | 1:15 | "Let's step through it." *(Next ▶ / click)* | Window filters bursts; fast promotion path untouched. Scenario buttons jump to a specific case. |
| **8** Trace method | 0:50 | "How do you even evaluate a metadata policy?" | Divide block# by fan-out → metadata trace; validated <0.01%. Reproducible. |
| **9** Results | 1:10 | "Does the window actually pay off?" | Best median & mean at every cache size — metadata **and** data. *(28.5% second, per-trace.)* |
| **10** Why it works | 0:55 | "The mechanism behind that number." | Promotes **<¼** as many blocks — and the genuinely hot ones. |
| **11** Production | 0:40 | "It ships in vSAN — miss ratio isn't everything." | Low overhead · scalable · dirty-aware · bounded · resizable. Simplicity. |
| **12** Takeaways | 0:40 | "To wrap up." | One correlation-aware idea → best miss ratio + deployed. Reproducible in libCacheSim. |

**Total = 11:30.** Running long → let the animation (7) and design (6) visuals carry them.
