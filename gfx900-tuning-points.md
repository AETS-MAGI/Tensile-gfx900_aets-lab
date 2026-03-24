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
