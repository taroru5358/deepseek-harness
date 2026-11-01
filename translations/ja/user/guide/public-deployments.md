# リバースプロキシの背後で Web UI を公開する

日本語 | [English](../../../../docs/user/guide/public-deployments.md) | [中文](../../../../docs/user/guide/public-deployments.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/public-deployments.md](../../../../docs/user/guide/public-deployments.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`dsh --profile web` は GUI をループバックポートで提供するため、別のマシンのブラウザからは到達できず、プロセスは実際に使うアドレスを何も知りません。前段のリバースプロキシがその外部区間（公開ホスト名、TLS、リスナーに転送する前に取り除くパスプレフィックス）を所有し、`--public-url` がブラウザの使うアドレスを DSH に伝えます:

```sh
dsh --profile web --public-url https://app.example/ui/ --trusted-host app.example
```

## `--public-url` が通知するもの

`--public-url` は、オプションのマウントプレフィックス付きの `http://` または `https://` ルートを受け付け、末尾が `/` になるように正規化します。印刷およびオープンされる起動 URL、`DSH_WEB_URL`、agent の runtime コンテキスト内の Web GUI URL を提供します。webserver はオリジンルートのルートを提供し続け、マウントを決して知りません。`publicUrl` 設定フィールドは同じ通知を公開します。

## プロキシが行う必要があること

- **ブラウザ向けの `Host` を保持する。** fence は受信した `Host` を受け入れ済みの authority と比較するため、プロキシはリスナーのアドレスに書き換えるのではなく、そのまま転送します。
- **マウントプレフィックスを取り除く。** リスナーはオリジンルートのルートに応答するため、`/ui/api/...` へのリクエストは `/api/...` として届く必要があります。
- **アップグレードを転送する。** ページが開くすべてのリクエストと WebSocket アップグレードは、`Upgrade` と `Connection` ヘッダーをそのままにしてリスナーに届く必要があります。HTTPS を提供する場合、TLS はプロキシの外部区間で終端します。
- **cookie スコープを書き換える。** プレーン HTTP を提供するバックエンドは、`Secure` なしの host-only `Path=/` cookie を発行します。プロキシは `Path` をマウント（`/ui/`）に書き換え、HTTPS 区間で `Secure` を追加します。自身の TLS を提供するバックエンドリスナーはすでに cookie を `Secure` とマークするため、プロキシは `Path` のみを書き換えます。
- **素のマウントをリダイレクトする。** `/ui/` が唯一のエントリです。これだけが起動トークンを session cookie と交換し、その cookie をすでに持つブラウザには `/ui/` または `/ui/index.html` がドキュメントを提供します。提供されるドキュメントは自身のディレクトリに対して URL を解決するためです。`/ui` へのリクエストは `/ui/` として届く必要があり、プレフィックスを取り除いたバックエンドはその外部パスを再構築できません。

## ブラウザが使う authority を信頼する

fence はループバック、リスナー自身のバインドアドレス、および `--trusted-host` が指定するすべての authority を受け入れます。ブラウザが他の authority でデプロイに到達すると、プロキシがどれほど正しくてもすべての API 呼び出しで 403 になります。そのためブラウザから見える authority を `--trusted-host` で指定してください。`--public-url` で通知するのは表示のみで、受け入れは行いません。ポートのないエントリは任意のポートに一致し、毎回異なるポートをバインドするトンネルに適しています。fence はリクエストを受け入れるだけで、印刷された URL の起動トークンと署名付き session cookie が認証します。TLS はどこで終端してもこれを変えません。証明書はサーバーをブラウザに対して識別し、ブラウザをサーバーに対して識別するものではありません。

通知される URL も fence もリッスンポート自体を保護しないため、ポートを信頼するプロキシまたはネットワークに制限してください。

## 外部区間を保護する

TLS はプロキシ、リスナー自身、または両方で終端します。`--public-url` は通知のままです。`http://` のブラウザ向けルートは起動トークンを平文で露出します。`https://` のプロキシルートはブラウザとプロキシ間の区間を保護します。プロキシとリスナー間の暗号化はリスナーの TLS 設定に依存します。

リスナーで終端するには、`--tls-cert` と `--tls-key` を渡します。それぞれプロセスの作業ディレクトリに対して解決されるパスです。証明書ファイルは完全なチェーンを保持し、キーファイルは暗号化されていない PEM 秘密鍵を保持します。DSH は証明書を取得、更新、監視せず、carrier はファイルを一度読むだけなので、変更するには置き換えてリスナーをリロードするかプロセスを再起動します。起動は欠落または無効なペアで失敗し、HTTP にフォールバックしません。ブラウザは自身のトラストストアと名前チェックを適用するため、証明書は開くホスト名または IP をカバーする必要があります。既定ポートは 3080 のままです。ネイティブ TLS にプロキシは不要ですが、プロキシデプロイと `--public-url` は独立してサポートされたままです。TLS リスナーは session cookie を `Secure` とマークします。ブラウザがそれらの cookie を返せるよう、公開のブラウザ向け区間は HTTPS のままにしてください。

印刷された URL はプロセスのクレデンシャルを運ぶため、意図したユーザーにのみ共有してください。

[Web アプリリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md#public-deployments) は `--public-url`、`--trusted-host`、`--tls-cert`/`--tls-key` コマンドラインオプションを説明します。Web profile では、`publicUrl` が `web-runtime` 行を設定し、`trustedHosts` が [Connection 行](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/connection/README.md) を設定し、`tls` が [webserver 行](../../subsystems/web-server.md#config) を設定します。
