---
kind: upgrade-guide
description: "Session の working directory が組み込み cwd 設定を置き換え、tool presentation メタデータはネストされた PTC 呼び出しでも実行される。"
---

# working-directory 設定と tool presentation の移行

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/working-directory/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/working-directory/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/working-directory/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/working-directory/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

次のリリースでは、ループの組み込み `{{cwd}}` prompt 変数と、`@deepseek-ai/dsh-subagent-acp` および `@deepseek-ai/dsh-subagent-dsh-sdk` の `cwd` 設定キーが削除される。未解決の `{{cwd}}` を保持するカスタム prompt テンプレートは、モデルリクエストの前に失敗する。Session の現在の実行ディレクトリは必須のユーザー context として提供され、元の `Session.header.cwd` とは別に永続化される。

新しい subagent は、提供されていれば開始リクエストの `cwd` を使い、そうでなければ親の現在のディレクトリを捕捉する。既存のプロセスは独自のディレクトリを保持する。実行ディレクトリを変更しても、元のプロジェクトは変わらず、追加の書き込み権限も付与されない。

tool の `output.presentationMeta` は、成功したネストされた PTC 呼び出しでも実行される。projection がネイティブのトップレベル呼び出しのみを想定している作者が影響を受ける。projection エラーはそのネストされた呼び出しを `ToolOutputError` で失敗させる。canonical な program 値とモデル content は変わらない。

## 移行

1. home、profile、または呼び出しの `personaSuffix` 設定と preset の `suffix` フィールドで、`{{cwd}}` を参照するディレクトリ句を削除し、無関係なテキストは残す。空になった suffix は省略するかクリアする。[working-directory サービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/working-directory/README.md) がディレクトリ context を提供する。
2. profile パッチ内の ACP と DSH-SDK の subagent provider 行から `config.cwd` を削除する。1つの子にディレクトリを選択するには、その [開始リクエスト](../../../subsystems/subagent.md) で `cwd` を渡す。親の実行ディレクトリを変更するには、子を開始する前に `working_directory({ cd: "/absolute/project" })` を使う。
3. 影響を受ける Session で `working_directory({})` を実行し、報告されたディレクトリを確認する。override なしで子を開始し、その実行ディレクトリを確認する。子が別のチェックアウトを使う必要がある場合は、明示的な開始 `cwd` を使う。`Session.header.cwd` は現在のディレクトリ操作ではなく、元のプロジェクト identity または権限 root のためにのみ読む。
4. カスタムの `output.presentationMeta` projection を純粋に保ち、その引数と canonical な結果から JSON 安全なメタデータを返す。ネイティブ呼び出しとネストされた PTC 呼び出しの両方を試し、記録されたメタデータが結果を変えずに操作の実際の target を識別することを確認する。

Dynamic context provider は、pre-step listener と route 準備の後、リクエスト admission 時に再度評価される。step ごとに1回の評価に依存せず、現在の状態を返す。登録済み context はその provider 内で変換する。admission refresh はキャッシュされたテキストを置き換えつつ、assembly 専用の context 寄与と、受け入れられた system prompt、tool schema、変数を保持する。
