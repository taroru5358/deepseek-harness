# prompt から永続 plugin を設定する

日本語 | [English](../../../../../docs/user/develop/practice/dynamic-cordis.md) | [中文](../../../../../docs/user/develop/practice/dynamic-cordis.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/practice/dynamic-cordis.md](../../../../../docs/user/develop/practice/dynamic-cordis.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Standard モードと Creator モードは [Plugin Manager](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md) を提供します。Creator モードは読み取り専用の [runtime inspection](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/tool-cordis/README.md) を追加します。plugin 設定は現在の profile に属し、その session に影響し、プロセス再起動後も残ります。

## MCP サーバーを接続する

Web profile を起動し、Creator モードを選択します。`ping` を公開する到達可能な Streamable HTTP MCP サーバーがある状態で、実際のエンドポイントを使ってこの prompt を送ります:

> この profile の `<endpoint>` にある MCP サーバーを `demo` として設定してください。その tool を今すぐ利用可能にし、ping tool を呼び出して結果を教えてください。

agent は `@deepseek-ai/dsh-mcp-client` を挿入する patch を持つ設定専用の bundle を書き込み、`plugin_manager install_bundle` でインストールします。HMR が有効な場合、tool は同じ実行中の session に現れます。管理結果（`application: applied`）と成功した `mcp__demo__ping` 呼び出しの両方を確認してください。`restart-required` の保存済みエントリはまだアクティブになっておらず、失敗したエントリは設定の修復が必要です。

設定を編集する前に bundle patch を読んでください。Plugin Manager を使ってエントリを無効化するか、bundle を削除します。受け入れられる設定と接続失敗の動作は [MCP client リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) を参照してください。
