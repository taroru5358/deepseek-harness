# サードパーティの memory MCP サーバーを接続する

日本語 | [English](../../../../docs/user/guide/mcp-memory.md) | [中文](../../../../docs/user/guide/mcp-memory.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/mcp-memory.md](../../../../docs/user/guide/mcp-memory.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

これら3つの**既定でオフのリファレンス設定**は、1つの memory システムを [`@deepseek-ai/dsh-mcp-client`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md) 経由で DSH に接続します。1つを選ぶか、同じ汎用 MCP 行を別のサーバーにコピーしてください。

これらのサードパーティ設定は相互運用性の例としてのみ提供されます。含まれていることは、DeepSeek による推奨、推薦、提携、継続的なサポートを意味しません。

## DSH が行うこと

DSH は選択された Cordis オーバーレイを解析し、設定された stdio コマンドを起動するか、設定された Streamable HTTP URL に接続し、MCP tool を検出して `mcp__<serverName>__<tool>` として公開します。DSH はサーバーのダウンロード、データベースの初期化、モデルや embedding provider の選択、クラウドアカウントの作成、ベンダーデータの移行、別個の HTTP サービスの監督を**行いません**。stdio の場合、汎用クライアントが DSH plugin ライフサイクルとともに子プロセスを起動・停止します。HTTP の場合、上流サービスがすでに実行されている必要があります。

stdio ブリッジは、子プロセスを起動する前に、名前が通常クレデンシャルを識別する環境変数とすべての `DSH_*` 変数を意図的に削除します。その他の環境変数は継承されたままです。各例は必要な最小限のオーバーライドのみを追加します。オプションの上流機能が別のシークレットを必要とする場合は、YAML にシークレットを直接置くのではなく、その変数をその行の `config.env` に追加してください。

## 1つ選ぶ

| システム | テスト済みピン | トランスポート | 上流の前提条件 |
|---|---:|---|---|
| [Memorix](https://github.com/AVIDS2/memorix) | `memorix@1.3.0` (`500792cad3144142293bfbb20acb4841c9f7fcfa`) | stdio | Node 22.18+ と `npm install --global memorix@1.3.0` |
| [MCP Reference Memory](https://github.com/modelcontextprotocol/servers/tree/main/src/memory) | `@modelcontextprotocol/server-memory@2026.7.4` (`6dd0a683e198783e30feabf7abaf42f925bd18b1`) | stdio | `npm install --global @modelcontextprotocol/server-memory@2026.7.4` |
| [Engram](https://github.com/Gentleman-Programming/engram) | `v1.20.0` (`ba9e46ced152c37a7cb9e576153c41995873e2fc`) | stdio | Go 1.25.10+ と `go install github.com/Gentleman-Programming/engram/cmd/engram@v1.20.0`、または対応するリリースバイナリ |

## 1つを有効にする

1つのオーバーレイを DSH に渡します:

```sh
dsh web --patch "$PWD/apps/cli/config/examples/mcp-memory/memorix.cordis.yml"
```

ファイル名を `mcp-reference-memory.cordis.yml` または `engram.cordis.yml` に置き換えます。パスはディスク上の任意の場所にあるコピーしたファイルを指してもかまいません。同梱の構成に memory サーバーは存在しないため、`--patch` を省略すると3つすべてが無効のままです。

選択を実行間で維持するには、選択したファイルの単一の `insert` patch をユーザー patch レイヤーにマージします。1つの profile なら `$DSH_HOME/profiles/<name>/cordis.patch.yml`、マシン上のすべての profile なら `$DSH_HOME/cordis.patch.yml` です。既存ファイルを上書きコピーしないでください。無関係なユーザー patch がすでに含まれている可能性があります。

## provider のセットアップ

### Memorix

```sh
npm install --global memorix@1.3.0
dsh web --patch "$PWD/apps/cli/config/examples/mcp-memory/memorix.cordis.yml"
```

Memorix は LLM や embedding サービスなしでローカルヒューリスティックモードで動作します。オプションの provider は Memorix 自身の `~/.memorix/config.toml` またはプロジェクトの `memorix.toml` で設定します。この例は DSH の作業ディレクトリから Memorix の Git プロジェクト ID を保持し、Memorix 自身の `~/.memorix/data` 既定値を使います。上書きするには DSH を起動する前に `MEMORIX_DATA_DIR` を設定します。

### MCP Reference Memory

```sh
npm install --global @modelcontextprotocol/server-memory@2026.7.4
dsh web --patch "$PWD/apps/cli/config/examples/mcp-memory/mcp-reference-memory.cordis.yml"
```

このリファレンスサーバーはローカル知識グラフを保存し、entity、relation、observation、read、search、open の tool を公開します。モデルや embedding サービスは不要です。この例は JSONL を、インストールされた npm パッケージディレクトリではなく `$HOME/.dsh-mcp-reference-memory.jsonl` に保存します。上書きするには DSH を起動する前に `MEMORY_FILE_PATH` を設定します。

検索は entity 名、型、observation に対する大文字小文字を区別しない部分文字列一致であり、セマンティック検索ではありません。サーバーは embedding、自動要約、競合解決、忘却ポリシーを追加しません。

### Engram

```sh
go install github.com/Gentleman-Programming/engram/cmd/engram@v1.20.0
dsh web --patch "$PWD/apps/cli/config/examples/mcp-memory/engram.cordis.yml"
```

Engram はストレージとプロジェクト選択を所有します。既定で `~/.engram` を使い、DSH の作業ディレクトリから Git プロジェクトを検出し、`ENGRAM_DATA_DIR` または `ENGRAM_PROJECT` を環境変数オーバーライドとして受け付けます。

## オプションの共有モデル指示

サーバーの tool 説明だけでは memory の使用が確実にトリガーされない場合は、この短くベンダー中立な指示を既存のモデル指示に追加します:

> ユーザーが何かを覚えておくように求めたら、memory 書き込み tool を呼び出してください。過去の情報が関連する可能性がある場合は、memory を検索し、関連する結果を使ってください。

これは追加のガイダンスにすぎません。これらの例は DSH のシステムプロンプトのペルソナを置き換えません。

## 書き込み、新しい session での想起、使用を検証する

1つの一意な値を使い、検証中は provider のストレージスコープを変更しないでください:

1. DSH session A で、次のように尋ねます: `Remember that my validation drink is lapsang-<unique suffix>.` モデルが provider の書き込み tool を呼び出し、tool が成功を返したことを確認します。
2. 同じ実行中の Host で DSH session B を作成します。session A の会話をコピーしないでください。次のように尋ねます: `What is my validation drink? Check memory.` モデルが provider の検索または想起 tool を呼び出し、値を返したことを確認します。
3. 引き続き session B で、次のように尋ねます: `Use that preference to suggest one drink for the meeting.` 回答が想起された値を使っていることを確認します。

新しい DSH session が必要です。Host の再起動は不要です。クラッシュした MCP 子プロセスはバックオフ付きの自動再接続と tool の再同期をトリガーします。tool はリストされたままで、呼び出しは停止中のみ失敗します。再接続の予算を使い切ると tool は登録解除され、リロードまたは再起動まで再接続は停止します。初回検出は非同期なので、最初の検証 prompt を送る前に provider の `mcp__...` tool を待ってください。

## 別の MCP サーバーを持ち込む

同じエントリフィールドをコピーし、一意の `id` と `serverName` を使います:

```yaml
- insert:
    - id: memory-my-server
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: my-memory
        transport: stdio
        command: my-memory-mcp
        args: []
        env: {}
        cwd: !!js process.cwd()
```

リモートサーバーの場合は、代わりに `transport: streamable-http`、`url`、`headers` を使います。provider 固有のインストール、ID、認証、モデル、embedding、永続化、ライセンスは provider の責任のままです。
