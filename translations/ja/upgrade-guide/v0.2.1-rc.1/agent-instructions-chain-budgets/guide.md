---
kind: upgrade-guide
description: "@deepseek-ai/dsh-agent-instructions の maxBytes 予算は各 instruction chain を個別に上限するため、両方の chain を保持する baseline は設定サイズの2倍に達することがある。"
---

# `@deepseek-ai/dsh-agent-instructions` の `maxBytes` を instruction chain ごとに設定する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-agent-instructions` の `maxBytes` は、完全な workspace-instruction baseline と各 reconciliation メッセージを上限していた。現在は user-global chain（`$DSH_HOME/AGENTS.md` と共有 agents root）と project chain を別々に上限するため、両方の chain を保持するメッセージは設定サイズの2倍に達することがある。エクスポートされる `loadBaselineInstructions` と `renderAgentInstructions` も同じ上限に従う。`maxBytes` を合計 prompt 予算として設定していたデプロイは、より大きな最大値を観測する。

## 移行

1. 以前の合計を維持する必要がある場合は、デプロイの `cordis.yml` またはパッチの `@deepseek-ai/dsh-agent-instructions` 行で `maxBytes` を半分にする。`dsh-base` は `65536` を維持する。`maxBytes` は baseline identity の一部であるため、変更後に resume される各 session は1つの完全な replacement baseline を受け取る。

   ```yaml
   - name: '@deepseek-ai/dsh-agent-instructions'
     config:
       maxBytes: 32768
   ```

2. 新しい session を開始し、session log で `source.kind` が `agent-instructions` である最初の `user/message` を読む。そのテキストは `maxBytes` の2倍以内に収まり、`Workspace instruction budget ...` 行が省略または切り捨てられたすべてのファイルを列挙する。
