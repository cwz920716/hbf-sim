# hbf-sim: Phase 1 plan

Status: **DRAFT, awaiting approval.** No simulator code is written until this plan is approved.

Source: the survey doc *High Bandwidth Flash for LLM Inference: Survey and Open Gaps*
(as of 2026-11-09). Section and table names below refer to that doc.

## 1. The claim

> For LLM decode with weights stored in HBF, prefetch with double buffering hides flash read
> latency whenever the lookahead time is at least the read latency plus the transfer time:
>
>     T_look >= tR + S / B
>
> where `tR` is the flash sensing latency, `S` is the bytes needed for the next unit of work
> (a layer, or a set of experts), and `B` is the effective read bandwidth.
> Dense models meet this easily. MoE models break it, because the experts to load are known
> only once the router runs, unless routing is predicted early enough and accurately enough.

The prototype must **support or refute** this statement. There are two parts:

- **(A) Dense:** with prefetch on, throughput is within a small margin of the all-in-fast-memory
  upper baseline everywhere the inequality holds. It falls off where the inequality fails.
- **(B) MoE:** if routing is known only at the layer itself, `T_look` collapses to the gate's
  compute time and the inequality fails across most of the published `tR` range. Predicting
  routing `L` layers ahead with accuracy `p` restores it, and the map shows which `(L, p)` are
  enough.

### Dimensionless form (axes of the break-even map)

| Ratio | Definition | Meaning |
| --- | --- | --- |
| Lookahead ratio `λ` | `T_look / (tR + S/B)` | `λ >= 1` means latency can be hidden |
| Bandwidth ratio `β` | `(S/B) / T_compute` | `β <= 1` means flash can keep up in steady state |
| Slowdown `σ` | `T_step / T_step(all-resident)` | the outcome; 1.0 is perfect |

The expected result is `σ ≈ 1` in the region `λ >= 1, β <= 1`, `σ ≈ β` where `λ >= 1, β > 1`,
and a latency penalty that grows with `(1 - λ)` where `λ < 1`. The simulator exists to check
this expected result, and to find where queueing, page granularity and finite parallelism make
the real boundary differ from the formula.

## 2. Model parameters

All flash parameters are **uncalibrated**: no HBF silicon exists, and every published figure is a
spec target or a simulation assumption. Every config file and plot will carry an `uncalibrated`
label until Phase 4 replaces a value with a measurement.

| Parameter | Baseline value | Sweep range | Source | Status |
| --- | --- | --- | --- | --- |
| Stack capacity | 512 GB | fixed (checks the model fits only; not used in timing) | Spec table; Ma and Patterson Table 3 | uncalibrated |
| Read bandwidth per stack `B` | 1.6 TB/s (1,638 GB/s) | 0.4, 1.6, 3.0 TB/s (the three spec grades) | Ma and Patterson Table 3; spec bandwidth grades | uncalibrated |
| Read latency `tR` | 10 µs | 1, 3, 10, 30 µs, plus 100 µs as a stress point | Device characteristics (tR of about 1 to 30 µs, DASH); Table 3 ("1,000s" ns) | uncalibrated |
| Read latency distribution | lognormal around `tR`, CV 0.2 | CV 0 (fixed), 0.2, 0.5 | none published; my assumption | uncalibrated, assumption |
| Page size (read granularity) | 4,096 B | 4 KB, 16 KB | Ma and Patterson Table 3 (bytes per read) | uncalibrated |
| Max outstanding page reads `Q` | `ceil(B * tR / page)` (just enough for full bandwidth) | 0.25x, 1x, 4x that value | none published; derived from Little's law | uncalibrated, assumption |
| Fast memory (HBM) bandwidth | 1.6 TB/s | fixed | Ma and Patterson Table 3 (HBM4) | uncalibrated |
| Prefetch depth `d` | 1 unit (double buffering) | 0 (off), 1, 2, 4 | design variable | n/a |

Notes:

- The survey's **Caveats** section records that papers use 1.6 TB/s while the spec grades span
  0.4 to 3.0 TB/s. That is why bandwidth is swept, not fixed.
- `Q` is the parameter most likely to move results and the least supported by sources. Reaching
  1.6 TB/s with `tR` = 10 µs and 4 KB pages needs about 3,900 page reads in flight. If the
  conclusion flips across the `Q` sweep, that is a finding, and it triggers a stop rule (section 3).
- **Workload shapes** are synthetic and parametric. A model is described by its layer count,
  bytes per layer (dense), and for MoE the expert count `E`, active experts `k`, bytes per expert,
  and the gate/attention time before the experts run. Compute time per layer is set by a roofline:
  `max(FLOPs / peak, bytes / B_fast)`. The reference shapes (one dense ~70B-class, one large
  fine-grained MoE) will be taken from the published model configs and cited in `docs/notes.md`
  before they are used. They are not filled in from memory here.

## 3. Exit criteria

The thresholds are proposed and need approval. The 10% and 20% figures are my choice, not taken
from any paper.

**Validity gates.** All must pass before any sweep result is trusted:

1. `tR = 0` with flash bandwidth equal to HBM bandwidth matches the all-resident baseline within 1%.
2. Prefetch off matches the closed-form on-demand time `Σ (tR + S/B + T_compute)` within 1%.
3. With latency CV = 0 and large `Q`, the simulated break-even boundary sits at `λ = 1` within
   one sweep step.

**Success: the claim is supported if all three hold.**

- **S1 (dense).** At the baseline parameters with prefetch depth >= 1, `σ <= 1.10` at every grid
  point with `λ >= 1` and `β <= 1`. Prefetch off is measurably worse (`σ >= 1.10`) at `tR >= 10 µs`.
- **S2 (MoE, no prediction).** With routing known only at the layer, `σ >= 1.20` for `tR >= 10 µs`
  at the baseline bandwidth.
- **S3 (MoE, prediction).** Some reachable `(L, p)`, with `L` = 1 to 2 layers ahead and `p <= 0.9`,
  brings `σ <= 1.10`. The map shows the boundary.

**Stop: write up what we have and do not proceed to Phase 4 if any of these hold.**

- **X1 (no problem to solve).** MoE with no routing prediction already gives `σ < 1.05` across the
  whole published `tR` and bandwidth range. Latency does not matter, and the gap is closed.
- **X2 (conclusion rests on an unknown).** S2 or S3 flips between true and false across the
  plausible `Q` or latency-distribution range. We would need real device numbers first, and
  building more model would not help.
- **X3 (simulator adds nothing).** Results match the closed-form expression everywhere to within
  2%. The result is then a formula and a plot, not a simulator, and GPU calibration is not
  worth doing.

The claim is **refuted** if S1 fails with the validity gates passing, or if S2 fails while X1 does
not apply. That outcome is recorded as the result. The plan is not adjusted after the fact.

## 4. Scope of version one

**In:**

- Decode phase only, one accelerator, one or more HBF stacks treated as one bandwidth pool.
- Weights in HBF. Everything mutable (KV cache, activations) is assumed to be in fast memory,
  with no cost charged.
- An event-driven simulator with page-granular reads, a read-latency distribution, a bandwidth
  cap, a cap on outstanding reads, prefetch depth, and double buffering.
- Two baselines: all-resident in fast memory (upper), and on-demand loading with no prefetch (lower).
- A dense sweep and an MoE routing sweep, producing the break-even map in `(λ, β)` and the
  MoE `(L, p)` map.
- Batch size 1 as the main case. An optional batch sweep only changes the number of unique experts
  touched per layer.

**Out, deliberately:**

- **Write endurance and wear**, and any writes to HBF (KV cache tiering is gap 2, not this one).
- **Real flash measurements.** Optional in Phase 4 only.
- **Power and thermals.** Published per-stack power figures disagree by about 2x anyway.
- **Reliability effects:** ECC latency, read disturb, refresh, retention.
- **FTL, garbage collection, and physical layout** across dies, planes and channels. Parallelism
  is folded into `B` and `Q`.
- **Interconnect detail** (UCIe, a direct path versus routing through the HBM base die, as in DASH).
- **Prefill, multi-accelerator serving, and request scheduling.**
- **Model accuracy effects** of quantization or bit errors.

## 5. Deliverables of Phase 2 (for reference, built only after approval)

| Item | Form |
| --- | --- |
| Simulator core | Python 3 package, standard-library event queue (`heapq`), NumPy for sampling, seeded RNG |
| Config | one YAML or TOML file per sweep; every flash value tagged `uncalibrated` |
| Tests | `pytest` for the three validity gates and the limiting cases |
| Sweeps | scripts that write CSV results |
| Plots | matplotlib: the break-even map in `(λ, β)` and the MoE `(L, p)` map |
| CI | GitHub Actions: tests, plus a reduced sweep on every push |

## 6. Open questions for approval

1. Are the exit thresholds right: 10% for "hidden", 20% for "broken", `p <= 0.9` and
   `L <= 2` for "reachable"?
2. Is Python with no simulation framework acceptable, or do you prefer SimPy or another language?
3. Batch 1 only, or should batch size be a first-class sweep axis in version one?
4. Which public models should anchor the reference shapes? I propose one ~70B-class dense model
   and one large fine-grained MoE, with shapes cited from their published configs.
