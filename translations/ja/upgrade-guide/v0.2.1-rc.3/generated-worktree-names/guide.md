---
kind: upgrade-guide
description: "生成される worktree 名は worktree- プレフィックスと完全な UUID サフィックスを dsh- と Session 由来の名前に置き換える。"
---

# 生成される worktree 名の consumer を更新する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.2.1-rc.3 では、`create_worktree` または `ctx.worktrees.create` で `name` を省略すると、デフォルトで `worktree-<UUID>` が生成される。次のリリースでは `dsh-<Session id からの最初の8桁の UUID 数字>-<4桁のランダムな16進数字>`、たとえば `dsh-6b1f02c4-9a3e` を生成する。UUID でない Session id には `dsh-<8桁のランダムな16進数字>` が付く。

`worktree-` プレフィックスや完全な UUID サフィックスを必要とするスクリプトと ignore ルールは更新が必要である。明示的な名前、設定された checkout pool、既存の checkout ディレクトリとブランチは変わらない。`namePrefix` 設定は引き続き利用できるが、プレフィックスのみを制御し、サフィックスの形式は制御しない。

## 移行

1. 生成名を認識するスクリプトと ignore ルールを更新する。古いパターンから再構築するよりも、返された `path` と `branch` フィールドを優先する。[worktree サービスリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/worktree/README.md#use-this-package) がこれらのフィールドを説明している。
2. 古いプレフィックスを維持するには、既存の `@deepseek-ai/dsh-experimental-worktree` 設定行で `config.namePrefix` を `worktree-` に設定する。完全な UUID サフィックスは復元されない。統合が特定の完全な名前を必要とする場合は、明示的な `name` を提供する。
3. `name` を省略して worktree を作成し、返された `branch` と `path` を調べ、更新された automation がそれを認識することを確認する。既存の worktree に名前変更やデータ移行は不要である。
