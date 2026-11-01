---
kind: upgrade-guide
description: "CUA の相対座標は 0 から 999 の整数を受け付け、以前の endpoint 1000 を拒否する。"
---

# CUA の相対座標は 0–999 を使う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`cua` tool と application/window オブジェクトメソッドは、`relative: true` の場合に 0 から 999 の整数を受け付けるようになった。以前の範囲は 0 から 1000 だった。座標は引き続き1000分率を使う。1000 で割り、window の寸法にスケールし、window 内に clamp する。0 から 999 の値は既存の位置を維持する。これは `click`、`hover`、`scroll`、および両方の `drag` endpoint に影響する。絶対的なスクリーンショットピクセルと AX element index は変わらない。

## 移行

1. 相対座標を生成するスクリプトと prompt を更新する。0 から 999 の既存値は変更しない。古い endpoint 1000 を 999 に置き換える。たとえば有効な古い座標には `Math.min(oldValue, 999)` を使う。より大きな window では、999 が以前の endpoint 1000 より内側に着地することがある。
2. 新しい window 状態を読み、0–999 の整数座標で影響を受けるアクションを呼ぶ。新しい観測で結果を確認する。相対値 1000、負の値、小数はネイティブ入力の前に拒否される。
