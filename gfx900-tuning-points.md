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

Integrated link status update:

- `g4-fallback-dispatch-link-check.sh` now runs fallback(strace) + dispatch(rocprofv3)
  in one condition and emits a unified gate summary.
- Latest tinyllama run reports:
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `direct_rocblas_or_tensile_dispatch=0`
  - `link_status=indirect_link_only_same_scenario`
- Latest qwen2.5:7b run reports:
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `direct_rocblas_or_tensile_dispatch=0`
  - `link_status=indirect_link_only_same_scenario`
- Interpretation:
  - Catalog-read + dispatch evidence is now co-captured on two models.
  - Direct Tensile dispatch evidence is still the remaining gate.

Layer/trace granularity update:

- `ROCm-MI25-build/g4-rocblas-layer-sweep.sh` now sweeps
  `ROCBLAS_LAYER=1,8,9,15,63` for the same workload.
- On both tinyllama and qwen2.5:7b:
  - layer `8` alone produced no trace lines
  - layer `1/9/15/63` stayed at handle-only logs (`rocblas_create_handle`)
  - GEMM/backend-like lines remained zero
- Practical setting is fixed to `ROCBLAS_LAYER=9` (trace + internal),
  and the next gate is workload-side path discovery, not layer tweaking.

Workload-side gate update:

- High-density sweep on `qwen2.5:7b` / `deepseek-r1:14b`
  (`NUM_PREDICT=512`, `long|math|code`) stayed at:
  - `direct_rocblas_or_tensile_dispatch=0`
  - `link_status=indirect_link_only_same_scenario`
  - summary: `g4_workload_path_sweep_20260324_023631.txt`
- `gpt-oss:latest` single-case probe reached:
  - `direct_rocblas_or_tensile_dispatch=1`
  - `rocblas_trace_gemm_lines=1002`
  - `kernel_tensile_like_rows=167`
  - summary: `g4_link_summary_gpt-oss_latest_20260324_024249.txt`

Interpretation update:

- "Dispatch evidence exists but direct Tensile link is missing" is no longer true globally.
- We now have one confirmed direct-dispatch anchor (`gpt-oss`), while
  tinyllama/qwen/deepseek remain indirect under current GGUF paths.
- This supports a two-track workflow:
  1) keep lightweight GGUF models for stability/regression checks
  2) use `gpt-oss` to maintain direct rocBLAS/Tensile visibility for low-level tuning.

Shape-priority update:

- Added shape extraction helper:
  - `ROCm-MI25-build/summarize-rocblas-gemm-shapes.sh`
- Current top shapes (from `gpt-oss` trace):
  - `512x512x2880`
  - `4096x512x64` / `64x512x4096`
  - `2880x512x4096`, `4096x512x2880`
- These shapes become the first candidates for Tensile-side inspection.

Formal reflection (fact / interpretation / implication):

1. Fact
   - `gpt-oss:latest` produced repeated `rocblas_gemm_ex` and
     `rocblas_gemm_tensile_backend` evidence in the same scenario.
   - `direct_rocblas_or_tensile_dispatch=1` is now observed.
2. Interpretation
   - tinyllama/qwen/deepseek stayed indirect, while another workload reached direct naming.
   - The dominant factor is likely workload/path shape, not trace-layer toggles.
3. Implication
   - A direct-dispatch probe baseline now exists for Tensile-side verification.
   - Next comparisons should anchor on this case when evaluating model/precision/path changes.
