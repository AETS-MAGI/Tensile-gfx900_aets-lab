# Tensile gfx900 Tuning Points (MI25)

Last updated: 2026-03-24
Target: `ROCm-repos_AETS/Tensile`

## 1. Scope

Tensile is currently the most relevant low-level area for gfx900 tuning,
because fallback catalog/hsaco asset reads are confirmed in runtime traces.

Important caveat:

- Current `fallback_confirmed` evidence is catalog-read level.
- Dispatch-level confirmation is still pending.

## 2. Priority order

### P0: Stable type-level catalog-read map

Goal:

- Track fallback asset reads at `Type_*` granularity.

Checkpoints:

- Per-run counts for `Type_HH`, `Type_HS_HPA`, etc.
- Type distribution differences by model (tinyllama vs qwen2.5:7b)

Deliverables:

- Type count table with run IDs
- `fallback_dat_openat` / `fallback_hsaco_openat` trend

### P1: Dispatch boundary evidence

Goal:

- Separate catalog load from actual kernel selection/execution.

Minimum target:

- At least one case linking workload shape to dispatch-side evidence.

### P2: Lazy-loading and startup cost contribution

Goal:

- Estimate how much catalog initialization affects TTFT.

Observed premise:

- `keep_alive=10m` strongly improves TTFT.
- Cold-start overhead is likely significant.

Checkpoints:

- Cold vs warm fallback open patterns
- First-run gap under `keep_alive=0s` vs `10m`

### P3: Asset focus list before kernel edits

Goal:

- Focus on actively used assets before touching kernel code.

Rule:

- Do not delete assets first.
- Build a "priority observation list" from actual access evidence.

## 3. Link to current benchmark findings

- tinyllama and qwen2.5:7b both benefit from `keep_alive=10m`.
- tinyllama throughput can improve with `num_thread=6`.
- Cross-model conservative baseline remains `safe + keep_alive=10m + num_thread=4`.

## 4. Current decision

- Next practical task: dispatch evidence + type map refinement.
- Delay kernel implementation changes until evidence is complete.

## 5. New timing split evidence (2026-03-24)

Instrumentation added in `ROCm-MI25-build`:

- `g4-fallback-strace-check.sh` (timestamped strace + optional rocBLAS trace)
- `summarize-fallback-phases.sh` (pid-level timing summary)

Latest tinyllama probe:

- `fallback_phase_summary_tinyllama_latest_20260324_014707.tsv`
- observed split:
  - fallback `.dat`: span `0.035186s`
  - fallback `.hsaco`: span `1.302794s`

Interpretation:

- `.dat` still looks like initialization burst.
- `.hsaco` spans a wider window and is a better dispatch-side candidate signal.
- This is still not final dispatch proof by itself, but it narrows the search area.

Update from rocprofv3 probe:

- `kernel_trace.csv` dispatch evidence is now available on tinyllama runs.
- Current dispatch names are predominantly ggml-hip kernels (`mul_mat_q`, `mul_mat_vec_q`,
  `flash_attn_*`, `quantize_*`) and do not explicitly expose Tensile naming.
- So, while "dispatch evidence exists" is now true, "Tensile dispatch directly confirmed"
  remains open and should be treated as the next gate.
