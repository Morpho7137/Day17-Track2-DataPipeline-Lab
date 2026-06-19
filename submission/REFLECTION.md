# Reflection - Day 17 (<= 200 words)

1. The most silent failure is the trace-to-dataset step. If I mis-map successful vs failed turns, the eval set and DPO pairs still look valid but train the wrong behavior. I would detect it by comparing split counts, prompt overlap, and manual spot checks on a sample of traces.

2. If I skip decontamination, the model can memorize eval prompts and overstate quality. The metric leak shows up as a better offline score than the real holdout, then a drop when the same prompts are removed from training.

3. A dangerous feature is current account balance in fraud or credit scoring. If I join the latest balance instead of the balance at event time, the row can include future activity and inflate performance.

4. The graph answers multi-hop questions like which returnable product ships from Hanoi and which policy clause applies. Graph is overkill for a single policy lookup, where flat retrieval is enough.
