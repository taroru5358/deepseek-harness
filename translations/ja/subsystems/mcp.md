# MCP

日本語 | [English](../../../docs/subsystems/mcp.md) | [中文](../../../docs/subsystems/mcp.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/mcp.md](../../../docs/subsystems/mcp.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

Model Context Protocol（MCP）は、外部サーバーが提供するツールにモデルを接続する。設定された各サーバーは、キャンセル、権限チェック、記録された結果、サポートされる画像出力を備えた通常の harness ツールを提供する。共有ツールは、サーバーが呼び出し元のスコープで設定されているときにリソースを発見・読み取り、サーバーの指示は記録されたシステム・プロンプトに加わる。公式 SDK は最新のプロトコル・リビジョン、またはサポートされるレガシー・リビジョンをネゴシエートする。このリファレンスは MCP パッケージ・グループの責務、スコープ、コンポジションの選択肢を扱う。サーバー設定は [client README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) が所有する。

## 目次

- [設定](#configuration)
- [責務とスコープ](#responsibilities-and-scope)
- [プロトコルと結果](#protocol-and-results)
- [リソースと指示](#resources-and-instructions)
- [リソース・プロバイダーの型](#resource-provider-types)
- [制限](#limits)
- [参考](#further-reading)

-----

<a id="configuration"></a>
## 設定

MCP サーバーはオプトインである。意図した Cordis スコープ内で、サーバーごとに 1 つの `@deepseek-ai/dsh-mcp-client` エントリーを設定する。同梱のすべてのプロファイルは[ツール・レジストリ](tools.md)を提供し、共有リソース・サービスを 1 回マウントする。ユーザーはクライアント・エントリーだけを設定する。可視の設定済みサーバーを持たない呼び出し元は、ネイティブ・モードでも PTC モードでも MCP のプロンプト・テキストやツールを受け取らない。

| 選択肢 | 設定の所有者 |
|---|---|
| サーバーのアイデンティティ、ローカル・プロセスまたは HTTP エンドポイント、資格情報、プロセス環境 | [クライアント設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md#use-this-package) |
| ツールとリソースの要求タイムアウト、起動失敗ポリシー、再接続 | [クライアント設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md#use-this-package) |
| リソースの発見と読み取り | [MCP リソース・サービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-resources/README.md#use-this-package)が同梱プロファイルに含まれる。設定フィールドはない |
| サーバー指示のサイズ制限 | クライアントの `maxInstructionBytes`。コンポジションが[システム・プロンプトのアセンブリ](system-prompt.md)を提供する |
| 権限の判断とサポートされる画像出力 | [ツール実行](tools.md)と[添付](attachment.md) |

プロトコルのネゴシエーションは SDK がサポートするリビジョンに従う。プロトコル・リビジョンを強制する製品設定はない。受け入れられるクライアント・フィールドとデフォルトは[設定カタログ](../../../docs/config-catalog.md#deepseek-aidsh-mcp-client)に一覧がある。

-----

<a id="responsibilities-and-scope"></a>
## 責務とスコープ

クライアントはサーバーごとの接続プラグインであり、harness ツール・レジストリのコンシューマーである。共有の `ctx.mcp` サービスは公開しない。外部サーバーが MCP 操作を実装し、SDK がプロトコル交換を所有し、クライアントが発見したツールを harness の実行に適合させる。

`mcp-resources` は共有リソース・ツールを所有し、呼び出し元のスコープでプロバイダーを選択する。各 MCP クライアントは自身の接続を通じてリソース操作を提供する。スコープ内の最初のプロバイダーがローカル共有ツールを有効化し、最後のものが削除されるとそれらも削除される。継承されたプロバイダーは引き続き可視である。サービスはこれらのツール登録を、いずれか 1 つのクライアントから独立して所有する。可視のクライアント・エントリーがアクティブである限り、接続障害によって共有リソース・ツールが削除されることはない。

設定された `serverName` は、その登録スコープ内でサーバーを識別する。そのスコープ内の 2 つのエントリーは同じ名前を予約できない。別々の Agent スコープは同じ名前を再利用できる。公開ツール名は設定されたサーバー名を含むため、異なるサーバーの同名ツールは区別されたままになる。登録エフェクトが名前と発見されたツールを所有する。プラグインの破棄は接続を閉じ、その寄与を削除する。

[native Cua Driver プロバイダー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/computer-use-cua-driver-native/README.md)は、MCP 接続を開かずにクライアントのエクスポートされた結果アダプターを共有する。デスクトップ・プロバイダーの選択は [computer-use サブシステム](computer-use.md)に属する。

-----

<a id="protocol-and-results"></a>
## プロトコルと結果

stdio と Streamable HTTP の両方が、公式 SDK のネゴシエーション、発見、プロトコル検証、キャンセルを使う。ツール・リストの変更は、レガシー通知またはモダンなサブスクリプションを通じて発見をトリガーする。更新に失敗した場合は前のツール世代が保持される。接続の回復は[クライアントのライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md#use-this-package)に従う。

結果アダプターは、プログラム的な呼び出し元のために正規の MCP JSON を保持し、通常のツール・コンテンツを準備する。サポートされる画像は添付システムを使い、サポートされないリッチ・コンテンツは明示的なテキスト診断を生成する。ポリシー違反と置き換えられた結果については、ツール・レジストリが引き続き権威を持つ。記録と最終的な提示は[ツール契約](tools.md)が所有し、MCP 固有の投影の詳細は[クライアント結果リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md#use-this-package)が所有する。

-----

<a id="resources-and-instructions"></a>
## リソースと指示

リソース呼び出しには、明示的に設定されたサーバー名が必要である。システム・プロンプトのアセンブリが利用可能な場合、リソース・サービスはディスパッチに使うのと同じレジストリから呼び出し元可視の名前を一覧する。ツールも指示も持たないサーバーも含まれる。共有レジストリはディスパッチ前に呼び出し元 Agent のスコープでその名前を解決する。利用できないサーバーはネットワーク要求なしで失敗する。発見と読み取りはオンデマンドで、ツールなしでリソースを公開するサーバーも対象になる。ページネーションとコンテンツのレンダリングは[リソース・パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-resources/README.md)が所有する。その生成されたツール・スキーマは[ツール・カタログ](../../../docs/tool-catalog.md#deepseek-aidsh-mcp-resources)にある。

リソース・プロバイダーは接続が所有したままである。スコープの破棄は登録を削除する。MCP クライアントはキャンセルと回復を制御する。正規の結果はプログラム的な呼び出し元のために完全な JSON を保持する一方、テキスト投影はバイナリ・ブロブを説明で置き換える。返されたテキストは通常のツール履歴に入る。サーバーが接続したという理由だけでコンテンツが取得されることはない。

システム・プロンプトのアセンブリが構成されているとき、クライアントは空白でないサーバー指示を、スコープ付きでサーバー帰属のセクションとして公開する。指示はリテラル・テキストのままで、公開前に設定されたサイズ制限を通過する。置き換えの接続は、発見が成功した後にのみ指示を公開する。指示がない場合はセクションを追加しない。[システム・プロンプト・サブシステム](system-prompt.md)がアセンブリと記録を所有する。

-----

<a id="resource-provider-types"></a>
## リソース・プロバイダーの型

接続プロバイダーは、1 つの操作と元のツール実行（呼び出し元とキャンセル・シグナルを含む）を受け取る。

```ts type-equiv
/** One supported resource operation, with server-owned cursors and URIs. */
type McpResourceRequest =
  | { method: 'resources/list' | 'resources/templates/list'; cursor?: string }
  | { method: 'resources/read'; uri: string }
```

```ts type-equiv
/** One configured server's resource access, owned by its MCP connection plugin. */
interface McpResourceProvider {
  /**
   * Run an operation against one live connection generation.
   * @param request - MCP resource method and parameters.
   * @param exec - caller identity and cancellation for this invocation.
   * @returns the protocol result as lossless JSON.
   */
  request(request: McpResourceRequest, exec: ToolExecution): Promise<JsonValue>
}
```

-----

<a id="limits"></a>
## 制限

MCP プロンプト・テンプレート、人間入力の elicitation、タスクベースの実行、リソース・サブスクリプションはサポートされない。リソース・ツールには呼び出し元可視の設定済みサーバーが必要である。バイナリ・リソースはモデル向けのテキスト説明を伴うプログラム的データのままになる。ツール機能を持たないサーバーは空のツール・セットで接続する。接続と発見のタイムアウトは SDK に従う。クライアントにそれら専用の設定はない。

-----

<a id="further-reading"></a>
## 参考

- [MCP パッケージ・グループ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/README.md) — パッケージのエントリー・ポイント。
- [MCP リソース](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-resources/README.md) — 共有ツールとリソース・プロバイダーのセマンティクス。
- [リソース可視性リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-resources/README.md) — 共有プロファイルのマウントと設定済みサーバーからの可視性。
- [サードパーティ製メモリ・サーバー](../user/guide/mcp-memory.md) — 製品設定ガイド。
- [プロトコル・ネゴシエーション・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) — SDK の所有権と互換性の決定。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxmcpresources--mcpresourceruntime"></a>

### `ctx.mcpResources` — `McpResourceRuntime`

スコープ付きリソース・アクセスと、設定済み MCP サーバーが共有する 3 つのツール。

```ts cordis-catalog
/**
 * Register one server and expose resource tools while that scope has providers.
 * @param server - configured server name, unique in this scope.
 * @param provider - connection-owned resource operations.
 * @returns the effect disposer for this exact registration.
 */
register(server: string, provider: McpResourceProvider): () => void
```

ソース: [`packages/mcp/mcp-resources/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-resources/src/index.ts)
<!-- END GENERATED cordis-surface -->
