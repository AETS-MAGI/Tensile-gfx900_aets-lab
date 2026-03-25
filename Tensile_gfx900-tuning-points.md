# Tensile gfx900 Tuning Points (MI25)

Last updated: 2026-03-25
Target: `ROCm-repos_AETS/Tensile`

## 1. Scope

Tensile is currently the most relevant low-level area for gfx900 tuning,
because fallback catalog/hsaco asset reads are confirmed in runtime traces.

### 1.1 Overlap topics with rocBLAS note (explicit list)

The following topics currently appear in both notes and are intentionally
tracked as shared context:

1. anchor workload (`gpt-oss` anchor condition)
2. baseline/side lane (`num_batch=512/1024`)
3. `keep_alive` threshold and observability
4. shape observability (top-shape families and lane shifts)
5. runtime path / mixed stack facts
6. direct/indirect dispatch wording and gate status

### 1.2 Role split (authoritative owner)

For maintenance going forward, this Tensile note is the primary location for:

- fallback asset coverage and catalog-read map
- HSACO candidate mapping/extraction and artifact narrowing
- disassembly signal summaries (`dot4`/`packed`/memory-like signals)
- queue-based kernel candidate narrowing from observed shapes

The following topics are secondary here and primary in
`../rocBLAS/rocBLAS_gfx900-tuning-points.md`:

- runtime path / mixed stack resolution order
- GEMM shape observation as a runtime tuning lane
- client/runtime knob behavior (`num_thread`, `num_ctx`, `num_predict`, `keep_alive`)
- baseline/side observability operations

### 1.3 Wording guard (for README and cross-note consistency)

- `direct dispatch`:
  same-scenario evidence where rocBLAS/Tensile-named dispatch-level traces are visible.
- `indirect link only`:
  fallback + dispatch are both confirmed in the same scenario, but direct
  rocBLAS/Tensile naming is absent.
- `catalog-read evidence` and `dispatch evidence` are different gates and must
  not be treated as interchangeable.

Important caveat:

- Current `fallback_confirmed` evidence is catalog-read level.
- `gpt-oss` anchor has direct dispatch confirmation, while tinyllama/qwen/deepseek
  remain mostly indirect on the current GGUF path.

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

## 6. Anchor-shape sweep entrypoint (2026-03-24)

To move from "single proof" to repeatable comparison, the main-node probe stack
now supports runtime knob sweeps with fixed shape targets.

Updated runtime knobs across G4 scripts:

- `NUM_CTX`
- `NUM_BATCH`
- `NUM_THREAD`
- `KEEP_ALIVE`

New sweep script:

- `ROCm-MI25-build/g4-gptoss-anchor-shape-sweep.sh`

Role in Tensile-side workflow:

- keeps anchor condition stable (`gpt-oss:latest`, `ROCBLAS_LAYER=9`)
- runs case matrices through the same link gate
- records per-case hit counts for current top-priority shapes:
  - `512x512x2880`
  - `4096x512x64`
  - `64x512x4096`
  - `2880x512x4096`
  - `4096x512x2880`

Practical implication:

- We can rank runtime settings by shape-level evidence density before touching
  Tensile assets, then narrow low-level tuning to shapes that remain dominant.

Validation snapshot (main-node, 2026-03-24):

- run:
  - `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=8192 NUM_BATCH_LIST=512 KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_033556.txt`
- key metrics:
  - `direct_hits=1`
  - `kernel_tensile_like_rows=167`
  - target hits:
    - `512x512x2880=192`
    - `2880x512x4096=96`
    - `4096x512x2880=96`

Additional batch comparison (`num_batch=512,1024`):

- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_033756.txt`
- both cases kept direct-dispatch evidence, but shape family moved with `N`:
  - `512x1024x2880`
  - `2880x1024x4096`
  - `4096x1024x2880`
- implication for Tensile-side prioritization:
  - maintain shape candidates as batch-conditioned families, not one fixed list.

## 7. Canonical anchor freeze (baseline vs side)

Canonical profile reference:

- `ROCm-MI25-build/ROCm-MI25-tips/G4_gptoss_anchor_profile.md`

Baseline lane (default):

- `MODEL=gpt-oss:latest`
- `ROCBLAS_LAYER=9`
- `NUM_CTX=8192`
- `NUM_BATCH=512`
- `NUM_PREDICT={64,128,256}`
- summary: `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_034636.txt`
- stable outcomes:
  - `direct_hits=3/3`
  - `kernel_tensile_like_rows=167` (all cases)
  - dominant `*x512x*` shape family

Side lane (shape-shift sensitivity):

- `NUM_BATCH=1024` + 1024-target shapes
- summary: `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_035250.txt`
- stable outcomes:
  - `direct_hits=3/3`
  - `kernel_tensile_like_rows=167` (all cases)
  - dominant `*x1024x*` shape family

Operational rule:

- Use baseline lane for main tuning decisions.
- Use side lane to check whether observed behavior survives batch-conditioned shape migration.

## 8. Single-knob sweep result: `num_ctx` under baseline512

Run:

- `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=4096,6144,8192,12288 NUM_BATCH_LIST=512 TARGET_SHAPES='512x512x2880,2880x512x4096,4096x512x2880' KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_040223.txt`

## 9. Anchor-lane aggregate check (2026-03-25)

Added aggregate helper (main-node workflow):

- `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-g4-anchor-lanes.sh`

Latest aggregate outputs:

- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_anchor_lane_status_gpt-oss_latest_20260325_010009.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_anchor_lane_status_gpt-oss_latest_20260325_010009.tsv`

Observed:

- baseline lane (`num_batch=512`): `ok_cases=5`, `direct_hits=5`
- side lane (`num_batch=1024`): `ok_cases=3`, `direct_hits=3`
- stream phase-window compare rows:
  - all rows keep `direct/fallback/dispatch = 1`
  - all rows keep `decode_signature_detected`

Implication:

- `prefill / decode` proxy split remains stable under baseline/side lane comparison.
- Current next step stays the same: prioritize shape-conditioned Tensile investigation
  after runtime/path evidence is fixed.

Observed (all 4 ctx cases):

- `direct_rocblas_or_tensile_dispatch=1`
- `kernel_tensile_like_rows=167`
- top-3 shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested range, `num_ctx` does not materially move the baseline512
  shape-family observation.
- The next sensitive knobs are likely `num_thread`, `keep_alive`, prompt profile, and `num_predict`.

## 9. Single-knob sweep result: `num_thread` under baseline512

Run:

- `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=8192 NUM_BATCH_LIST=512 NUM_THREAD_LIST=2,4,6,8 TARGET_SHAPES='512x512x2880,2880x512x4096,4096x512x2880' KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_040941.txt`

Observed (all 4 thread cases):

- `direct_rocblas_or_tensile_dispatch=1`
- `kernel_tensile_like_rows=167`
- top-3 shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested range, `num_thread` does not materially move the baseline512
  shape-family observation.
- Next knobs should move to prompt profile and extended `num_predict` ranges.

## 10. Single-knob sweep result: prompt profile under baseline512

Run setup:

- baseline fixed: `MODEL=gpt-oss:latest`, `NUM_PREDICT=128`, `NUM_CTX=8192`,
  `NUM_BATCH=512`, `KEEP_ALIVE=5m`
- profiles: `short`, `long`, `code`, `math`
- compare note:
  - `g4_baseline512_prompt_profile_sweep_compare_20260324_042420.txt`

Observed (all 4 profiles):

- `direct_rocblas_or_tensile_dispatch=1`
- `kernel_tensile_like_rows=167`
- top-3 shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested profile set, prompt style/length does not materially move
  the baseline512 Tensile-observed shape family.
- The next single-knob target should be extended `num_predict` ranges.

## 11. Single-knob sweep result: extended `num_predict` under baseline512

Run setup:

- baseline fixed: `MODEL=gpt-oss:latest`, `NUM_CTX=8192`,
  `NUM_BATCH=512`, `KEEP_ALIVE=5m`
- `NUM_PREDICT={64,128,256,512,1024}`
- compare note:
  - `g4_baseline512_numpredict_sweep_compare_20260324_043625.txt`

Observed:

- `direct_rocblas_or_tensile_dispatch=1` for all 5 cases
- `kernel_tensile_like_rows=167` for all 5 cases
- top-3 shape hits unchanged for all 5 cases:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`
- generation-side evidence still scales:
  - `eval_count` follows requested decode length until stop condition
  - `1024` case ended at `eval_count=797` with `done_reason=stop`

Implication:

- Under baseline512, extended decode length did not materially change the
  observed Tensile-linked shape family in current trace mode.
- Next step should isolate prefill and decode observation windows.

## 12. Stream phase-window sweep (`num_predict=64..1024`) under baseline512

Run setup (main-node, 2026-03-24):

- command:
  - `NUM_PREDICT_LIST=64,128,256,512,1024 ./g4-stream-phase-window-sweep.sh`
- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.txt`
- table:
  - `ROCm-MI25-build/vega_path_check_logs/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.tsv`

Observed:

- all 5 cases succeeded (`ok_cases=5`, `failed_cases=0`)
- all 5 cases preserved link-gate signals:
  - `direct_rocblas_or_tensile_dispatch=1`
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
- all 5 cases showed the same phase-window proxy shape:
  - `phase_split_status_proxy=decode_signature_detected`
  - `prefill_kernel_tensile_like_rows=0`
  - `decode_kernel_tensile_like_rows=167`
  - `stream_first_token_channel=thinking`

Formal reflection (fact / interpretation / implication):

1. Fact
   - In the baseline512 + gpt-oss anchor, stream phase-window probes stayed
     `decode_signature_detected` across `num_predict=64..1024`.
   - Tensile-linked dispatch visibility stayed intact in every case.
2. Interpretation
   - Longer decode windows increase runtime but do not move the currently
     observed Tensile-linked signature under this anchor.
   - The first-token path remains thinking-channel-first for this workload.
3. Implication
   - This profile is now suitable as a stable observability lane for
     batch/path/model deltas in later Tensile-side comparisons.
   - Caveat remains: phase split is proxy-based and not strict token-level attribution.

## 13. Baseline512 vs side1024 in stream phase-window lane

Comparison inputs (main-node, 2026-03-24):

- baseline tsv:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.tsv`
- side1024 tsv:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_122317.tsv`
- derived compare table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_batch_compare_gpt-oss_latest_20260324_123206.tsv`

Observed:

- both lanes kept, for all `num_predict={64,128,256,512,1024}`:
  - `direct_rocblas_or_tensile_dispatch=1`
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `phase_split_status_proxy=decode_signature_detected`
  - `decode_kernel_tensile_like_rows=167`
- side1024 increased total stream wall time in every case
  while preserving the same stream-visible signature.

Formal reflection (fact / interpretation / implication):

1. Fact
   - Moving `num_batch` 512 -> 1024 did not alter the phase-window proxy class
     or dispatch gate outcomes in this lane.
   - Runtime wall-time scaled upward consistently.
2. Interpretation
   - In this observability setup, batch mostly scales runtime cost and does not
     trigger a different Tensile-linked signature class.
3. Implication
   - Maintain baseline512 as canonical.
   - Use side1024 as a secondary lane for robustness and scaling comparisons.

## 14. Stream observability sensitivity: `keep_alive` threshold

Runs (main-node, baseline512, `num_predict=128`):

- sweep A (`0s,5m,30m`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_123600.tsv`
- 0s recheck (2 runs):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_keepalive_0s_recheck_20260324_123825.tsv`
- sweep B (`1s,10s,30s,5m`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_123938.tsv`

Observed:

- `keep_alive=0s` and `1s` repeatedly showed:
  - `dispatch_confirmed=0`
  - `phase_split_status_proxy=unavailable`
  - rocprof summary with `trace_file_count=0`, `csv_file_count=0`
- `keep_alive=10s/30s/5m` consistently showed:
  - `dispatch_confirmed=1`
  - `phase_split_status_proxy=decode_signature_detected`
  - `decode_kernel_tensile_like_rows=167`

Formal reflection (fact / interpretation / implication):

1. Fact
   - Extremely short keep-alive values (`0s`, `1s`) reproducibly break
     rocprof phase-window visibility in this stream lane.
2. Interpretation
   - This reflects a measurement-window instability rather than a clear
     change in the underlying Tensile-linked path itself.
3. Implication
   - For stable Tensile-side stream observability, enforce
     `keep_alive>=10s` in canonical probe settings.

## 15. Cross-batch confirmation of `keep_alive>=10s`

Additional check (main-node, `num_predict=128`):

- side1024 (`num_batch=1024`) keep-alive sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_124412.tsv`
- baseline/side combined table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_batch_compare_gpt-oss_latest_20260324_124713.tsv`

Observed:

- same threshold pattern across both lanes:
  - `keep_alive=1s` -> `dispatch_confirmed=0`, `phase_split_status_proxy=unavailable`
  - `keep_alive>=10s` -> `dispatch_confirmed=1`, `decode_signature_detected`

Implication:

- The keep-alive threshold is lane-agnostic in current observations.
- Use `keep_alive>=10s` as a common prerequisite for stable Tensile-linked stream evidence.

## 16. Non-dot4 focus confirmation + shape-first queue (2026-03-25)

Inputs:

- rocBLAS gemm dtype summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_gemm_dtype_summary_rocblas_gemm_shapes_g4_rocblas_trace_gpt-oss_latest_20260324_045255_20260324_045658_20260325_010632.txt`
- fallback type summary (catalog-read side):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/fallback_type_summary_tinyllama_20260324.txt`

Observed:

- [main-node confirmed] In the current direct-dispatch anchor (`gpt-oss`),
  gemm rows are:
  - `non_dot4_like=501/501` (100%)
  - `int8_or_i32_like=0/501`
- [main-node confirmed] Catalog-read summaries still include int8-related types
  (`Type_4xi8I_HPA`, `Type_I8I_HPA`) together with HH/HS/HPA families.

Interpretation:

- [inference] In the current LLM anchor lane, the active dispatch-visible work is
  dominated by BF16/F16/F32 signatures, while some int8 families remain only as
  catalog-read evidence in this dataset.
- This supports the strategy of prioritizing non-dot4 shape families first,
  then revisiting int8 families only when direct dispatch evidence appears.

Shape-first queue for Tensile-side inspection:

1. Queue-A (baseline core, highest frequency)
   - `512x512x2880`
   - `2880x512x4096`
   - `4096x512x2880`
2. Queue-B (decode-tail candidates)
   - `512x93x2880`
   - `32x512x2880`
3. Queue-C (batched-ex sensitivity family)
   - `4608x512x64`
   - `64x512x4608`
   - `8192x512x64`
   - `64x512x8192`

Operational next step:

- Split per-shape observation notes by Queue-A/B/C and keep the same anchor
  settings (`gpt-oss`, baseline512, `ROCBLAS_LAYER=9`, `keep_alive>=10s`).
- Promote only stable findings to low-level asset/kernel inspection.

Update:

- Queue-A per-shape split is now available in rocBLAS-side notes:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x512x2880.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_2880x512x4096.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4096x512x2880.md`
- Queue-B/C shape families are now visible in full-shape prefill/full compare TSVs:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_shape_prefill_full_compare_g4_rocblas_trace_gpt-oss_latest_20260325_011553__g4_rocblas_trace_gpt-oss_latest_20260325_011629_20260325_012104.tsv`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_shape_prefill_full_compare_g4_rocblas_trace_gpt-oss_latest_20260325_011411__g4_rocblas_trace_gpt-oss_latest_20260325_011439_20260325_012104.tsv`
- Queue-B/C per-shape notes are now added as well:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x93x2880.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_32x512x2880.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4608x512x64.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x4608.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_8192x512x64.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x8192.md`

Update 2:

- Kernel candidate narrowing from rocprof summary pairs is now available:
  - helper: `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-kernel-candidates.sh`
  - baseline output:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150.tsv`
  - side output:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011425__rocprofv3_summary_gpt-oss_latest_20260325_011502_20260325_013150.tsv`

Observed [main-node confirmed]:

- both lanes keep visible `Cijk_*` entries and `mul_mat_*` families in the top
  candidate list while decode-heavy rows increase in full runs.

Implication [inference]:

- The "kernel candidate narrowing" gate is now covered enough to move to the
  next TODO: HSACO extraction target selection from `Cijk_*` candidates.

Update 3:

- HSACO mapping/extraction gate is now covered:
  - map helper: `/home/limonene/ROCm-project/ROCm-MI25-build/map-kernel-candidates-to-hsaco.sh`
  - extract helper: `/home/limonene/ROCm-project/ROCm-MI25-build/extract-hsaco-targets.sh`
- mapping result:
  - 4 `Cijk_*` candidates -> 3 matched to `*gfx900*.hsaco` (1 unmatched `...ISA900...`)
- extracted target set:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150_20260325_013452_20260325_013541`
  - `3 files / 876K`

Implication [inference]:

- Disassembly can start from a constrained 3-file scope instead of scanning all
  `*gfx900*.hsaco` assets.

Update 4:

- disasm signal summary on the extracted 3-file set is now available:
  - helper: `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-hsaco-disasm-signals.sh`
  - summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150_20260325_013452_20260325_013541_20260325_013821.txt`

Observed [main-node confirmed]:

- `dot4_positive_files=0`
- `mfma_positive_files=0`
- `packed_positive_files=1` (HH fallback)
- `memory_positive_files=3`

Implication [inference]:

- Current fallback-gfx900 candidate set appears to rely on FMA-like + memory
  access signatures, not dot4/mfma, under this probe.
- This is enough to mark the "instruction feature observation" gate as covered.

## 17. Observation-only deepening cycle (2026-03-25 02:23 JST)

Scope:

- No Tensile asset edit and no source patch in this cycle.
- Goal is only deeper observability:
  - top-shape re-observation in baseline/side lanes
  - prefill/full and stream-window split comparison
  - candidate -> hsaco -> disasm refresh under the same anchor

Anchor lane re-observation [main-node confirmed]:

- baseline summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022355.txt`
  - `direct/fallback/dispatch=1`, `gemm_lines=1002`
- side summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022435.txt`
  - `direct/fallback/dispatch=1`, `gemm_lines=1336`

Prefill/full proxy split [main-node confirmed]:

- baseline:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022531.txt`
- side:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022637.txt`
- both lanes:
  - `phase_split_status=prefill_dominant_signature`
  - `decode_delta_gemm_lines=0`
  - `decode_delta_target_shape_hits=0`

Stream phase-window split [main-node confirmed]:

- baseline sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022802.txt`
- side sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022953.txt`
- both lanes:
  - `ok_cases=3`
  - `decode_signature_cases=3`
  - all rows `direct/fallback/dispatch=1`
  - `decode_kernel_tensile_like_rows=167`

Candidate to HSACO refresh [main-node confirmed]:

- kernel candidates:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022651__rocprofv3_summary_gpt-oss_latest_20260325_022727_20260325_023315.txt`
- hsaco map:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022651__rocprofv3_summary_gpt-oss_latest_20260325_022727_20260325_023315_20260325_023336.txt`
- map status:
  - `total_candidates=4`, `matched_candidates=3`, unmatched `...ISA900...` persists
- disasm signal summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347_20260325_023354.txt`
  - `dot4_positive_files=0`, `mfma_positive_files=0`, `packed_positive_files=1`, `memory_positive_files=3`

Interpretation:

- [inference] Under the current anchor, observability remains stable without any
  low-level modification.
- [inference] Treat `prefill/full proxy` and `stream phase-window` as separate
  evidence layers to avoid over-claiming dispatch attribution.
- [inference] This closes another observation cycle and keeps the next action as
  shape-priority evidence refinement, not immediate code/asset rewrite.

## 18. Decode-side reproducibility rerun (2026-03-25 03:31-03:33 JST)

Scope:

- observation-only rerun (no Tensile asset edit, no source patch)
- fixed anchor: `MODEL=gpt-oss:latest`, `ROCBLAS_LAYER=9`
- lane comparison:
  - baseline (`NUM_BATCH=512`)
  - side (`NUM_BATCH=1024`)

Executed:

- phase-window sweeps:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_031811.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_032242.txt`
- prefill/full split checks:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_032955.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_033100.txt`

Observed [main-node confirmed]:

- phase-window sweeps:
  - baseline: `decode_signature_cases=5/5`
  - side: `decode_signature_cases=5/5`
  - both lanes keep:
    - `direct_rocblas_or_tensile_dispatch=1`
    - `fallback_confirmed=1`
    - `dispatch_confirmed=1`
    - `decode_kernel_tensile_like_rows=167`
- prefill/full split:
  - both lanes: `phase_split_status=prefill_dominant_signature`
  - both lanes: `decode_delta_gemm_lines=0`
  - both lanes: `decode_delta_target_shape_hits=0`

- side lane target-shape correction rerun:
  - summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_033458.txt`
  - corrected target set:
    - `512x1024x2880`, `2880x1024x4096`, `4096x1024x2880`
  - observed hits:
    - `shape_512_1024_2880=288`
    - `shape_2880_1024_4096=144`
    - `shape_4096_1024_2880=144`
  - `phase_split_status` remains `prefill_dominant_signature`

Interpretation [inference]:

- Decode-side signature is reproducible in the stream-window layer under the
  current anchor/batch settings.
- Prefill/full proxy remains prefill-dominant for this pair and should not be
  merged into the same gate as stream-window decode evidence.
- Catalog-read evidence and dispatch-side evidence remain separate by design;
  this rerun strengthens reproducibility, not a new low-level mechanism claim.

## 19. K1-entry HSACO/disasm parity check (2026-03-25 11 JST)

Scope:

- observation-only parity check for baseline/side lanes at HSACO/disasm layer.
- no Tensile source/asset rewrite in this step.

Executed [main-node confirmed]:

- baseline lane:
  - hsaco map:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092328__rocprofv3_summary_gpt-oss_latest_20260325_092357_20260325_110634_20260325_110635.tsv`
  - disasm summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092328__rocprofv3_summary_gpt-oss_latest_20260325_092357_20260325_110634_20260325_110635_20260325_110812_20260325_110821.txt`
- side lane:
  - hsaco map:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092433__rocprofv3_summary_gpt-oss_latest_20260325_092510_20260325_110636_20260325_110636.tsv`
  - disasm summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092433__rocprofv3_summary_gpt-oss_latest_20260325_092510_20260325_110636_20260325_110636_20260325_110812_20260325_110822.txt`

Observed [main-node confirmed]:

- both lanes:
  - `total_candidates=4`, `matched_candidates=3`
  - unmatched stays `...SB...ISA900...`
- disasm aggregate parity:
  - `total_files=3`
  - `dot4_positive_files=0`
  - `mfma_positive_files=0`
  - `packed_positive_files=1`
  - `memory_positive_files=3`

Interpretation [inference]:

- baseline/side parity is now confirmed through
  candidate -> hsaco -> disasm layers for the K1-entry set.
- This reduces entry ambiguity before any future shape-scoped low-level A/B step.

## 20. Runtime-path A/B impact on fallback evidence (2026-03-25 14 JST)

Scope:

- keep anchor fixed (`gpt-oss`, `NUM_BATCH=512`, `ROCBLAS_LAYER=9`).
- compare runtime path only via `ROCBLAS_TENSILE_LIBPATH`.
- no Tensile source/asset rewrite in this step.

Evidence [main-node confirmed]:

- AETS lane (`.../rocblas-install/lib/rocblas/library`):
  - fallback/dispatch/direct all observed in:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_link_summary_gpt-oss_latest_20260325_135852.txt`
  - key fallback counts:
    - `fallback_dat_openat=56`
    - `fallback_hsaco_openat=56`
- system lane (`/opt/rocm-7.2.0/lib/rocblas/library`):
  - strace raw:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/g4_strace_openat_gpt-oss_latest_20260325_140141.log*`
  - rocprof summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocprofv3_summary_gpt-oss_latest_20260325_140345.txt`
  - observed:
    - `fallback_dat_openat=0`
    - `fallback_hsaco_openat=0`
    - `dispatch_confirmed=0` (for this run)
- consolidated A/B table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_runtime_path_ab_compare_20260325_140536.tsv`

Tooling note:

- `g4-fallback-strace-check.sh` currently exits before summary emission when
  fallback matches are zero, so system-lane fallback counts were computed from
  raw strace logs.

Interpretation [inference]:

- For the fixed anchor, fallback catalog visibility is runtime-path sensitive.
- Keep catalog-read evidence and dispatch evidence separated; this section does
  not claim a strict kernel-level 1:1 mapping.

## 21. One-shape entry loop reflection (2026-03-25 14 JST)

Scope:

- consume one-shape K1 loop results for Tensile-side observability tracking.
- no Tensile source or asset edits.

Evidence [main-node confirmed]:

- loop outputs:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_loop_k1_entry_20260325_1shape.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_loop_k1_entry_20260325_1shape.tsv`
- target shape:
  - `512x512x2880`
- AETS lane:
  - `fallback_dat_openat=56`
  - `fallback_hsaco_openat=56`
  - `shape_target_hits=192`
- system lane:
  - `fallback_dat_openat=0`
  - `fallback_hsaco_openat=0`
  - `shape_target_hits=0`

Interpretation [inference]:

- For this one-shape anchor loop, fallback catalog visibility and shape
  observability remain aligned with runtime path selection.
- This section records path-level evidence only; strict dispatch-to-catalog
  1:1 mapping is still pending.

## 22. One-shape repeat consolidation (2026-03-25 14 JST)

Scope:

- consolidate 3 repeated one-shape runs under same anchor and lane split.
- no Tensile source or asset edits.

Evidence [main-node confirmed]:

- repeat summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_repeat_summary_k1_entry_20260325_1shape_20260325_143343.tsv`
- key lane stability:
  - AETS lane: `fallback_mode=1`, `dispatch_mode=1`, `direct_mode=1`, `shape_hits_mode=192`
  - system lane: `fallback_mode=0`, `dispatch_mode=0`, `direct_mode=0`, `shape_hits_mode=0`
  - all corresponding `all_same=1`

Interpretation [inference]:

- Catalog-read observability difference between lanes is repeat-stable for the
  one-shape anchor gate.
- Causality remains unexpanded (no 1:1 kernel mapping claim).

## 23. Single-knob control reflection (`num_predict 128 vs 256`) (2026-03-25 14 JST)

Scope:

- keep shape gate and lane split unchanged
- vary only `NUM_PREDICT` for control comparison
- no Tensile source/asset edits

Evidence [main-node confirmed]:

- compare TSV:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_control_compare_num_predict_128_vs_256_20260325_144910.tsv`
- lane invariants:
  - AETS: `decode_signature_detected`, `shape_hits_mode=192`
  - system: `unavailable`, `shape_hits_mode=0`
- fallback/gemm observability pattern remains lane-separated as before.

Interpretation [inference]:

- This control run keeps Tensile-side observability class unchanged while
  runtime cost metrics move with `NUM_PREDICT`.
- Catalog-read and dispatch evidence are still recorded as separate layers;
  strict kernel-level mapping is not newly claimed.

## 24. Single-knob control reflection (`num_thread 4 vs 6`) (2026-03-25 15 JST)

Scope:

- keep shape gate and lane split unchanged
- vary only `NUM_THREAD` for control comparison
- no Tensile source/asset edits

Evidence [main-node confirmed]:

- nt4 summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_repeat_summary_k1_entry_20260325_1shape_nt4_20260325_155820.tsv`
- nt6 summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_repeat_summary_k1_entry_20260325_1shape_nt6_20260325_155820.tsv`
- compare:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_control_compare_num_thread_4_vs_6_20260325_1559.tsv`

Observed [main-node confirmed]:

- lane invariants remain unchanged:
  - AETS: `decode_signature_detected`, `shape_hits_mode=192`
  - system: `unavailable`, `shape_hits_mode=0`
  - fallback/dispatch/direct class per lane is unchanged
- metric deltas (AETS) indicate lower latency and slightly higher throughput on
  `NUM_THREAD=6` vs `4`.

Interpretation [inference]:

- This control keeps the Tensile-side observability class stable while runtime
  metrics move.
- Catalog-read evidence and dispatch evidence remain separate layers; strict
  1:1 kernel-level mapping is still pending.

## 25. Single-knob control reflection (`keep_alive 5m vs 0s`) (2026-03-25 18 JST)

Scope:

- keep shape gate and lane split unchanged
- vary only `KEEP_ALIVE` for control comparison
- keep `NUM_THREAD=6` fixed
- no Tensile source/asset edits

Evidence [main-node confirmed]:

- ka5m summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_repeat_summary_k1_entry_20260325_1shape_ka5m_20260325_185004.tsv`
- ka0s summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_repeat_summary_k1_entry_20260325_1shape_ka0s_20260325_185004.tsv`
- compare:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_k1_single_shape_control_compare_keep_alive_5m_vs_0s_20260325_1850.tsv`

Observed [main-node confirmed]:

- AETS lane invariants partly hold:
  - `shape_hits_mode=192` remains unchanged
  - fallback/direct class remains `1`
  - but `dispatch_mode` changes `1 -> 0`
  - and `phase_set` changes to `unavailable`
- system lane remains unchanged (`unavailable`, shape hits `0`)

Interpretation [inference]:

- `KEEP_ALIVE=0s` improves runtime metrics in this control, but it does not
  preserve the full observability class used in the current gate.
- Keep catalog-read and dispatch evidence separated; this run should not be
  interpreted as a stronger kernel-level mapping result.
