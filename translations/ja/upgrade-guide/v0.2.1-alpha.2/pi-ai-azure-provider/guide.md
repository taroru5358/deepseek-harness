---
kind: upgrade-guide
description: "pi-ai の Azure provider route は azure-openai-responses から azure に改名される。"
---

# Azure provider route の改名

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

インストールされた pi-ai カタログは Azure provider を `azure-openai-responses` ではなく `azure` として識別する。`azure` route は Responses と Chat Completions の両方のモデルを含む。Responses API の識別子は `azure-openai-responses` のままで、`AZURE_OPENAI_*` 環境変数も名前を維持する。古い provider キーを使う設定とモデル選択は Azure カタログを解決しなくなる。

## 移行

1. `$DSH_HOME/profiles/<profile>/cordis.patch.yml` または別の供給レイヤーの `llm-pi-ai` 設定で、`config.providers.azure-openai-responses` を `config.providers.azure` に改名する。既存のフィールドと `apiKeyEnv` 参照を保持する。
2. `models` の下にはカタログの model ID を使う。カスタムの Azure deployment 名は `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` でマップする。混在カタログは、カタログにない model ID のプロトコルを推論できない。明示的な route レベルの `api` はその route のすべてのモデルを override し、Azure provider を選択するものではない。
3. 明示的な request/default provider の選択を `azure` に更新し、既存の Session を継続する場合は Azure モデルを再選択する。キーが古いエントリ用に保存されていた場合は、Settings → Models で新しい Azure エントリを設定する。履歴 Session event を編集しない。
4. Settings → Models が `azure` を提供し、期待するモデルを列挙することを確認し、設定した deployment を使って短いリクエストを送る。
