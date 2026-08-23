# kvstore

Encrypted per-key KV cache backend (LMCache or HiCache plugin). Host-RAM
tier plaintext, disk tier encrypted with a per-key DEK; tenant-salted
prefix hashes plus a shared namespace for the public preamble. Open
question: KDA recurrent-state offload support (docs/design/cohort.md
sections 6 and 7, review items 11-14).
