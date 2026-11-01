---
kind: upgrade-guide
description: "pi-ai 1.3.0 は provider プロトコル、reasoning の選択肢、履歴 replay、無効な tool 引数の処理を変更する。"
---

# pi-ai 1.3.0 のプロトコルと replay の変更

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

pi-ai の 1.1.0 から 1.3.0 へのアップグレードは、これらのデフォルトを変更する。明示的な profile override は引き続き優先される。

| route または動作 | 新しい動作 |
|---|---|
| `kimi-coding` | `/coding` の `anthropic-messages` の代わりに、`https://api.kimi.com/coding/v1` の `openai-completions` を使う |
| `opencode`, `opencode-go` | endpoint は `/zen` を `/inference` に置き換える。`/go` と `/v1` サフィックスはモデル固有のままである |
| Xiaomi MiMo カタログ（token-plan route を含む） | reasoning は `off` と `high` のみを提供する |
| `kimi-coding/kimi-for-coding-highspeed` | reasoning `off` を提供しない |
| Assistant の reasoning 履歴 | 送信リクエストは、別の provider、API、モデルの thinking をテキストに変換せず省略する。ネイティブメタデータのない劣化 replay も thinking を省略する |
| 無効な完了済み tool 引数 | 試行は `MALFORMED_RESPONSE` で失敗する。tool を実行せず、成功した replay 状態も公開しない。デフォルトの normal retry policy はこのコードを除外する |

保存済みの履歴は保持される。元のモデルに戻ると、その有効なネイティブ replay メタデータが復元される。トークン上限で切り捨てられた出力は、不完全な tool 呼び出しを引き続き破棄し、通常どおり継続できる。認証情報の移行は不要である。

## 移行

1. `cordis.patch.yml` の `providers.<provider>.baseURL`、`api`、`compat` override を見直す。upstream の Kimi Coding endpoint に従う override では、Completions プロトコルと `/v1` パスを使う。カスタム gateway は実際のプロトコルとパスを明示的に維持しなければならない。該当する場合は OpenCode の URL を `/inference` に更新する。カタログのデフォルトを意図する場合は override を省略する。
2. profile、preset、Session のサポートされない reasoning 選択を、現在のモデルメタデータが示すレベルに置き換える。
3. モデル変更をまたいで必要な事実は、ユーザーメッセージまたは通常の assistant 返信に置く。以前の private reasoning が可視テキストになることに依存しない。
4. `MALFORMED_RESPONSE` については、報告された無効な引数を調べ、モデルまたは gateway の出力を修正する。短い prompt と tool 呼び出しで影響を受ける route を検証する。[設定リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md) を参照。
