---
kind: upgrade-guide
description: "tool presentation は native または ptc を受け付け、混合の both モードは拒否される。"
---

# native または PTC の tool presentation を選択する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-tools` と `@deepseek-ai/dsh-agent-tool-presentation` の `mode` フィールドは `native` または `ptc` のみを受け付ける。`both` 値は拒否される。これは、出荷される profile パッチを通じて tools 設定を提供する `DSH_TOOLS_MODE=both` にも影響する。

`native` は可視 tool を直接の関数呼び出しとして公開する。`ptc` は `run_code` を公開し、プログラムは生成された SDK を通じて可視 tool を呼ぶ。異なる agent preset は同じアプリケーション内で引き続き異なるモードを使える。

## 移行

1. `cordis.yml`、profile の `cordis.patch.yml`、またはカスタム overlay で、`mode: both` を、直接呼び出しには `mode: native`、プログラムベースの呼び出しには `mode: ptc` に置き換える。
2. `DSH_TOOLS_MODE=both` を `DSH_TOOLS_MODE=native` または `DSH_TOOLS_MODE=ptc` に置き換える。preset 固有の選択には、代わりに preset の `@deepseek-ai/dsh-agent-tool-presentation` 行を更新する。
3. `ctx.tools.presentAs('both')` を呼ぶコードを、`native` または `ptc` を選択するよう更新する。PTC プログラムでは呼び出しを `run_code` 内に保つ。SDK 関数名を直接のモデル tool 呼び出しとして出力しない。
4. 影響を受ける profile を起動し、設定がロードされることを確認する。native リクエストは許可された直接 tool を公開し、PTC リクエストは `run_code` とその SDK 命令を公開する。
