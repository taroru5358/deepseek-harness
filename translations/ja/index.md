# DeepSeek Harness 日本語ドキュメント

> これは非公式の日本語訳です。英語版が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。
>
> コード、コマンド、イベント名、識別子は原文のまま残しています。Mermaid 図のラベルも、イベント名との対応を保つため原文のままです。

## 翻訳済みドキュメント

- [アーキテクチャ](architecture.md) — Cordis、profile / bundle、turn flow、session log、capability seam
- [エージェントのターンとステップのライフサイクル](agent-lifecycle.md) — turn / step のシーケンス図
- [用語集](glossary.md) — capability-seam、scope、goal、human command、turn / step / round / exchange、Ralph
- [ツール実行パイプライン](tool-execution-pipeline.md) — tool 呼び出しの前処理・承認・sandbox・結果確定

## 原文

- [README（日本語）](overview.md)
- [README（英語）](../../README.md)
- [docs/architecture.md（英語）](../../docs/architecture.md)
- [docs/agent-lifecycle.md（英語）](../../docs/agent-lifecycle.md)
- [docs/glossary.md（英語）](../../docs/glossary.md)
- [docs/tool-execution-pipeline.md（英語）](../../docs/tool-execution-pipeline.md)

## 翻訳の方針

- 上流の i18n 契約は、`docs/**` と README に対して en/zh の3ファイルペア（`foo.md` / `foo.zh.md` / `foo.i18n.yaml`）を要求し、locale ディレクトリを禁止しています。この翻訳はスキャン対象外の `translations/ja/` に置いた非公式訳で、上流の PR にはそのまま出しません。
- この翻訳は、既存の en/zh ペアリング（`.i18n.yaml`）には統合していません。`pnpm run verify-translation-pairing` の対象外です。
- 英語版を正とし、日本語版は参照用の非公式訳とする。
- コード、コマンド、イベント名、型名、設定キーは翻訳しない。
- `harness`、`turn`、`step`、`session` などの用語は、初出時に日本語の補足を付けつつ、原則として原文の用語を残す。
- 原文の更新に追従できるよう、翻訳ファイルは英語ファイルと分けて管理する。
