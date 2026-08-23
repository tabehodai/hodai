# bench

Agent-shaped load generator and sweep configs for benchmark day: 300-500
concurrent streams, think-act-wait loops (~30% decode duty), 20:1
prefill:decode, 16k/48k/128k/700k contexts, cold and warm cache, tier
eviction, recovery. Acceptance criteria: docs/design/cohort.md section 11
plus the review's item 5 list. Dated results land in results/.
