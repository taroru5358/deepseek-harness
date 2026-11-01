# Web UI を使う

日本語 | [English](../../../../docs/user/guide/index.md) | [中文](../../../../docs/user/guide/index.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/index.md](../../../../docs/user/guide/index.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[ルート README](../../overview.md#run) から Web UI を起動します。コマンドが URL を出力します。このガイドはそのサーバーが実行された後に始まります。`dsh` プロセスは呼び出し元のディレクトリを既定の filesystem の場所として使いますが、新しい Web UI には workspace を追加するまで選択された workspace がありません。

## モデルを設定する

**Settings → Models** を開き、[DeepSeek API キー](https://platform.deepseek.com/) を入力して保存します。モデルルートはサーバーを再起動せずにすぐ利用可能になります。

[モデル設定ガイド](providers.md) は他の provider とカスタム OpenAI 互換エンドポイントを扱います。

## workspace を選択する

**Choose workspace** をクリックし、`dsh` を起動したプロジェクトディレクトリを追加して選択します。workspace が選択されるまで session composer は利用できません。

## タスクを実行する

session を開始して次を送ります:

> このリポジトリを要約し、主要なパッケージを特定してください。

agent は workspace ファイルの読み取りと編集、コマンドの実行、作業の委譲、プランの維持ができます。Web UI は、アクティブな permission ポリシーの下で承認が必要な操作の前に確認します。

## 続ける

- [モデルを設定する](providers.md)
- [Python SDK を使う](python-sdk.md)
- [リバースプロキシの背後で Web UI を公開する](public-deployments.md)
- [他の CLI モードを使う](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/README.md)
- [plugin を開発する](../develop/basic/index.md)
