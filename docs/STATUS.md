# Current status

Apate is a research MVP undergoing hardening. It is not certified for hostile Internet deployment.

The implemented path is SSH → persistent worker → FUSE → a Redis namespace per session → constrained generation when an existing manifest-backed file is first read. Audit events feed PostgreSQL and the evidence collector.

The hardening review corrected generation ownership races, write/truncate corruption, shared blob deletion, cross-session filesystem sharing, uninitialized metadata access, broken gateway imports, watcher wiring, and deployment endpoint mismatches.

HTTP is removed. Rust Layer 0 is still present at `src/chronos/layer0` and remains disconnected. Dashboard and simulation components are experimental; their existence is not evidence that every panel or simulation behavior is correct.

See [Hardening review](HARDENING_REVIEW.md) for verification evidence and remaining risks. Older roadmap and architecture documents describe historical intentions and may disagree with the current implementation.
