# Tensile (gfx900/MI25 実験フォーク)

[English](README.md) | [日本語](README.ja.md)

## 概要

このリポジトリは `AETS-MAGI/Tensile-gfx900_aets-lab` として運用している、`ROCm/Tensile` の実験フォークです。

- 目的: gfx900/MI25 向け bring-up と ROCm ランタイム調査
- 運用方針: `main` を基準枝、実験は `gfx900-bringup` などの専用ブランチで実施
- ライセンス: upstream の `LICENSE` と著作権表記を維持

## 注意

`ROCm/Tensile` 自体は退役リポジトリ扱いで、現行の開発先は `ROCm/rocm-libraries` です。

## 参照

- upstream: https://github.com/ROCm/Tensile
- 現行統合先: https://github.com/ROCm/rocm-libraries
- 公式ドキュメント: https://rocm.docs.amd.com/projects/Tensile/en/latest/index.html

## 関連リポジトリ

- セットアップ/検証ワークスペース: https://github.com/AETS-MAGI/ROCm-MI25-build
- 連携する rocBLAS フォーク: https://github.com/AETS-MAGI/rocBLAS-gfx900_aets-lab

詳細な手順や追加情報は英語版 [README.md](README.md) を参照してください。
