# Clock2Q+ — Stage Cue Cards

One line to get in, one line to land. Full text in `SPEAKER_SCRIPT.md`; live version toggles with **S** in the deck.

| # | Time | ↪ Transition in | ★ Remember (land this) |
|---|------|-----------------|------------------------|
| **1** Title | 0:30 | Name + one-line; simple & deployed. | Simple, production idea — promise intuition. |
| **2** Hot path | 1:00 | "Where this lives." | Metadata cache = hottest path; overhead & simplicity are first-class, not just miss ratio. |
| **3** Correlated refs | 1:15 | "Metadata behaves differently." | Correlated bursts **look** hot but aren't — the pattern of the whole talk. |
| **4** Why others fail | 1:15 | "How do the best policies handle a burst?" | S3-FIFO over-promotes; Clock2Q under-promotes. Want both. |
| **5** Insight | 1:00 | "The insight — almost embarrassingly simple." | **Where** the re-hit happens is the signal. One-line change. |
| **6** Design | 1:30 | "Put the window into the full picture." | 4 paths: cold/burst→Ghost · hot→Main direct · Ghost-hit→Main · Main=second-chance. |
| **7** Animation | 1:30 | "Rather than a static trace — let's watch it." | Window silently filters bursts; fast path untouched. *(Let one scenario finish.)* |
| **8** Trace method | 1:00 | "How do you even evaluate a metadata policy?" | Divide block# by fan-out → metadata trace; validated <0.01%. Reproducible. |
| **9** Results | 1:15 | "Does the window actually pay off?" | Best median & mean at every cache size — metadata **and** data. *(28.5% second, per-trace.)* |
| **10** Why it works | 1:00 | "The mechanism behind that number." | Promotes **<¼** as many blocks — and the genuinely hot ones. |
| **11** Production | 0:45 | "It ships in vSAN — miss ratio isn't everything." | Low overhead · scalable · dirty-aware · bounded · resizable. Simplicity. |
| **12** Takeaways | 0:45 | "To wrap up." | One correlation-aware idea → best miss ratio + deployed. Reproducible in libCacheSim. |

**Total ≈ 12:00.** Running long → trim slide 6 (let animation carry it) and tighten slide 4.
