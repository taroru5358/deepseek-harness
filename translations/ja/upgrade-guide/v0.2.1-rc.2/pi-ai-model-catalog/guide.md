---
kind: upgrade-guide
description: "pi-ai 1.3.0 のカタログはモデル ID を削除し、一部のモデル容量を引き下げる。"
---

# pi-ai 1.3.0 のモデルカタログ変更

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

pi-ai の依存関係が 1.1.0 から 1.3.0 に変わる。以下のチャットモデル ID がカタログから外れる。保存済みの `modelOverrides` はカタログ欠落の診断を報告し、新しい override は拒否される。明示的な `models` エントリはカタログのデフォルトを失う。

| Provider | 削除されたモデル ID |
|---|---|
| `google-vertex` | `gemini-flash-latest`, `gemini-flash-lite-latest` |
| `nvidia` | `moonshotai/kimi-k3`, `nvidia/nemotron-3-super-120b-a12b` |
| `opencode` | `exo-free`, `fledge-alpha-free`, `ling-3.0-flash-fin-free`, `ling-3.1-flash-free` |
| `opencode-go` | `deepseek-v4-flash`, `deepseek-v4-flash-vision-exp`, `grok-4.6`, `minimax-m2.7` |
| `openrouter` | `anthropic/claude-sonnet-4`, `apodex/apodex-1.1-mini:free`, `inclusionai/ling-3.0-flash-fin`, `inclusionai/ling-3.0-flash-sante:free`, `qwen/qwen-plus-2025-07-28`, `qwen/qwen3-235b-a22b`, `qwen/qwen3-30b-a3b-thinking-2507`, `qwen/qwen3-8b`, `qwen/qwen3-coder-plus`, `qwen/qwen3-max`, `qwen/qwen3-max-thinking`, `qwen/qwen3-vl-235b-a22b-thinking`, `qwen/qwen3-vl-32b-instruct`, `qwen/qwen3-vl-8b-thinking`, `qwen/qwen3.6-max-preview`, `sao10k/l3.1-euryale-70b`, `unbiased/pareto`, `unbiased/pareto-26.10-preview` |
| `vercel-ai-gateway` | `alibaba/qwen-3.6-max-preview`, `alibaba/qwen3-235b-a22b-thinking`, `alibaba/qwen3-coder-plus`, `alibaba/qwen3-max`, `alibaba/qwen3-max-preview`, `alibaba/qwen3-vl-235b-a22b-instruct`, `alibaba/qwen3-vl-thinking`, `anthropic/claude-3-haiku`, `anthropic/claude-opus-4`, `inclusionai/ling-3.1-flash-free` |

以下のカタログ上限が引き下げられる。値はトークン数で、`—` はその上限が引き下げられていないことを示す。明示的な override は引き続き優先される。`maxTokens` はモデル容量を表し、暗黙のリクエスト上限ではない。

| Provider/model | 新しい contextWindow / maxTokens |
|---|---|
| `anthropic/claude-sonnet-4-5` | 200000 / — |
| `anthropic/claude-sonnet-4-5-20250929` | 200000 / — |
| `openrouter/deepseek/deepseek-v4-flash` | 1048575 / 943717 |
| `openrouter/inclusionai/ling-3.0-flash-vl` | 131072 / — |
| `openrouter/meta-llama/llama-4-scout` | 131072 / — |
| `openrouter/minimax/minimax-m2` | — / 131072 |
| `openrouter/minimax/minimax-m2.5` | 198000 / 32768 |
| `openrouter/minimax/minimax-m2.7` | — / 131072 |
| `openrouter/moonshotai/kimi-k2-thinking` | — / 98304 |
| `openrouter/nvidia/nemotron-3-ultra-550b-a55b` | 202800 / — |
| `openrouter/openai/gpt-oss-120b` | — / 65536 |
| `openrouter/qwen/qwen3-235b-a22b-thinking-2507` | 128000 / 16384 |
| `openrouter/qwen/qwen3.5-35b-a3b` | 256000 / 16384 |
| `openrouter/qwen/qwen3.8-2.4t-a95b` | 1000000 / — |
| `openrouter/qwen/qwen3.8-27b` | 262144 / — |
| `openrouter/~z-ai/glm-latest` | — / 131072 |
| `radius/claude-sonnet-4-5` | 200000 / — |
| `vercel-ai-gateway/alibaba/qwen3-coder` | — / 32768 |
| `vercel-ai-gateway/deepseek/deepseek-v4.1-flash-fast` | — / 32768 |

`vercel-ai-gateway` のモデル `alibaba/qwen-3-235b` は非 reasoning になった。プロトコルと effort の変更には[別のガイド](../pi-ai-protocol-replay/guide.md)がある。

## 移行

1. Settings → Models で、削除された ID をサポートされるモデルに置き換える。エンドポイント、capability、価格を確認する。ID に自動的な置き換えはない。
2. `cordis.patch.yml`、preset パッチ、影響を受ける Session の選択を更新する。不要になった `providers.<provider>.modelOverrides.<id>` エントリを削除し、`providers.<provider>.models` リスト内の削除済み ID を置き換える。[設定リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md) を参照。
3. endpoint がサポートする値に対して容量と reasoning の override を見直す。profile を起動し、モデルに設定診断がないことを確認して、短い prompt を送る。
