# hbf-sim: notes and decision log

This file records decisions and the state at the end of each session. Findings about HBF itself
go into the survey doc (*High Bandwidth Flash for LLM Inference: Survey and Open Gaps*), not here.

Format: newest first. Each decision says **what** was decided, **why**, and its **status**
(proposed, approved, or superseded).

---

## 2026-11-10: Phase 1 kickoff

### State at end of session

- Phase 0: repository created (README, BSD-2 license). Branch `ccr-d216175d-49nuf7`.
- Phase 1: `docs/plan.md` and this file are drafted. **Waiting for approval.**
- Phase 2: not started. No code exists.

### Decisions (all proposed, pending approval)

| # | Decision | Why | Status |
| --- | --- | --- | --- |
| D1 | The claim to test: prefetch hides `tR` when `T_look >= tR + S/B`, and MoE routing breaks this unless routing is predicted early | The survey's open gap 1, picked as the one most directly testable by simulation | proposed |
| D2 | State results in dimensionless ratios `λ = T_look/(tR + S/B)`, `β = (S/B)/T_compute`, and slowdown `σ` | Keeps conclusions valid while the uncalibrated device numbers change | proposed |
| D3 | Baseline device: 1.6 TB/s, `tR` = 10 µs, 4 KB pages, 512 GB per stack; all tagged uncalibrated | The middle of the published ranges (spec, Ma and Patterson Table 3, DASH) | proposed |
| D4 | Model flash parallelism as one cap `Q` on outstanding page reads, and sweep it | No published per-die or per-plane figures for HBF. Little's law gives about 3,900 reads in flight at baseline | proposed |
| D5 | Latency distribution: lognormal, CV 0.2 baseline | Placeholder assumption, no source. Swept to CV 0 and 0.5 | proposed |
| D6 | Exit criteria: S1 to S3 for success, X1 to X3 for stop (see plan section 3) | Fixed before any results, so the plan cannot be tuned to fit them | proposed |
| D7 | Version one leaves out endurance, writes, power, reliability, FTL, interconnect detail, prefill, real flash | Keeps version one to the latency-hiding question only | proposed |
| D8 | Python, `heapq` event queue, NumPy, matplotlib, pytest, GitHub Actions | Small, dependency-light, and runs in CI on every push | proposed |

### Back-of-envelope behind the claim (my estimates, not sourced)

These numbers explain why the dense case is expected to pass and the MoE case to be the
interesting one. They will be replaced with cited model shapes before use.

- **Dense, ~70B parameters at 8 bits, ~80 layers.** About 0.9 GB per layer, so about 0.55 ms to
  stream one layer at 1.6 TB/s. That is about 50x a 10 µs `tR`. One layer of lookahead hides
  latency easily, and `λ` is far above 1.
- **Fine-grained MoE expert of ~40 MB at 8 bits.** About 25 µs to transfer at 1.6 TB/s, which is
  the same order as `tR`. The router's output arrives only microseconds before the experts run,
  so `λ` falls below 1 without prediction.

### Open questions (to resolve at approval)

See `docs/plan.md` section 6: the exit thresholds, the language and framework, the batch-size
axis, and which public models anchor the reference shapes.

### Discrepancies to keep in view

- The bandwidth assumption differs: papers use 1.6 TB/s per stack, while the spec grades span
  0.4 to 3.0 TB/s. This is handled by sweeping all three grades.
- Read latency is "1,000s of ns" in Ma and Patterson Table 3, versus about 1 to 30 µs per DASH.
  These are consistent, and the sweep covers both, plus 100 µs as a stress point.
- The survey draws on abstracts and excerpts. Check a paper before quoting a specific number from it.
