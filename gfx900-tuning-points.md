# Tensile gfx900 Tuning Points (MI25)

最終更新: 2026-03-24
対象: `ROCm-repos_AETS/Tensile`

## 1. 位置づけ

Tensile 側は、現行 LLM 推論で **fallback catalog / hsaco の実アクセス証跡** が取れているため、gfx900 最適化の主戦場。

ただし現時点の `fallback_confirmed` は「catalog read」段階であり、`dispatch` 確認は未完。

## 2. 優先度つき調整点

## P0: 型別 catalog read の固定化

目的:

- fallback 資産アクセスを `Type_*` 単位で追跡可能にする

確認項目:

- run ごとの `Type_HH`, `Type_HS_HPA` などの出現回数
- モデル差（tinyllama / qwen2.5:7b）で型分布がどう変わるか

成果物:

- 型別カウント表（run_id 付き）
- `fallback_dat_openat` / `fallback_hsaco_openat` の推移

## P1: dispatch 境界の可視化

目的:

- 「catalog 読み込み」と「実際の kernel 選択・実行」を分離する

最小到達:

- 少なくとも1ケースで、呼び出しサイズと対応する実行側証跡を紐付ける

注記:

- ここが取れると `fallback_confirmed` の粒度が一段上がる

## P2: lazy-loading と startup コスト

目的:

- 初回遅延（TTFT）に効く可能性のある catalog 初期化コストを把握する

観測前提:

- `keep_alive=10m` が TTFT に有利（main-node confirmed）
- つまり cold start コストの寄与が大きい

確認項目:

- cold/warm で fallback 資産 open パターンがどう変わるか
- `0s` と `10m` での初回 run 乖離

## P3: kernel 資産の整理ルール

目的:

- 実際に使う資産と参照のみ資産を区別し、調査対象を絞る

方針:

- いきなり削除しない
- まずアクセス実績ベースで「重点観測リスト」を作る

## 3. 実測との接続（2026-03-24）

[main-node confirmed]

- tinyllama / qwen2.5:7b とも `keep_alive=10m` が TTFT を大幅改善
- tinyllama で `num_thread=6` は tok/s 優位、`4` はバランス

解釈:

- Tensile 側の改善は、cold-start 影響と分離して評価する必要がある
- 比較の既定条件は `safe + keep_alive=10m + num_thread=4`

## 4. 当面の判断

- 次の実作業は「dispatch 証跡化」と「型別マップ作成」
- kernel 実装改変は、その証跡が揃ってから着手
