---
kind: upgrade-guide
description: "説明のない pi-ai モデルは、provider が明示的なデフォルトを設定しない限り、より大きな context 容量と出力容量を継承する。"
---

# pi-ai のデフォルトモデル容量

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

容量が設定とインストール済みカタログの両方にないモデルでは、`llm-pi-ai` は `defaultContextWindow` を 262,144 から 1,000,000 トークンに、`defaultMaxTokens` を 32,768 から 128,000 トークンに引き上げる。想定 context window が大きくなると、自動 compaction が遅れることがある。明示的なモデル容量、カタログ値、provider のデフォルトは引き続き優先される。出力容量のフォールバックは、Harness のリクエストレベルの `maxTokens` デフォルトにはならない。

## 移行

1. カスタム endpoint の実際の上限を確認する。`cordis.patch.yml` の `llm-pi-ai` 設定で、上限が異なる場合は `providers.<provider>.defaultContextWindow` と `providers.<provider>.defaultMaxTokens` を設定する。以前のデフォルトを維持するには、それぞれ `262144` と `32768` を使う。
2. 同じ provider で上限の異なるモデルには、その `providers.<provider>.models` エントリで `contextWindow` と `maxTokens` を設定する。明示的に設定されたモデルの `maxTokens` は、Harness のリクエストデフォルトも提供する。
3. profile 設定を再度開き、保存された数値を確認する。[provider 設定リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md) を参照。
