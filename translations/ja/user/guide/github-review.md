# GitHub webhook からレビュー用 Session を作成する

日本語 | [English](../../../../docs/user/guide/github-review.md) | [中文](../../../../docs/user/guide/github-review.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/github-review.md](../../../../docs/user/guide/github-review.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このオプトインのオーバーレイは、署名付き GitHub エンドポイントを `dsh web` に追加します。設定されたリポジトリの pull request がドラフトからレビュー準備完了に変わると、ルールはリポジトリの Web Workspace の下にタイトル付きのルート Session を作成し、読み取り専用のレビュー prompt を開始します。

## 前提条件

- DSH が Web Workspace として登録してよいローカルチェックアウト。
- `DSH_GITHUB_WEBHOOK_SECRET` クレデンシャル参照を通じて利用できる高エントロピーの GitHub webhook シークレット。
- 1つの公開 URL をループバックリスナーに転送できる TLS リバースプロキシまたはトンネル。
- コンテンツタイプ `application/json` での Pull requests イベントへの GitHub webhook サブスクリプション。

オーバーレイは Workspace を起動ディレクトリに、リスナーを `127.0.0.1:3081` に既定設定します。`DSH_GITHUB_REVIEW_WORKSPACE` と `DSH_GITHUB_WEBHOOK_PORT` で上書きします。

## DSH を起動する

シークレットを生成し、再起動間で同じ値を保持します:

```sh
export DSH_GITHUB_WEBHOOK_SECRET="$(openssl rand -hex 32)"
printf '%s\n' "$DSH_GITHUB_WEBHOOK_SECRET"
```

まず両方の実験的パッケージを Web profile にインストールします。profile は peer を自動インストールしません。GitHub アダプターは webhook runtime を明示的に必要とします:

```sh
dsh plugin --profile web add @deepseek-ai/dsh-webhook @deepseek-ai/dsh-webhook-github
export DSH_GITHUB_REVIEW_WORKSPACE=/path/to/deepseek-harness
dsh web --patch "${DSH_HOME:-$HOME/.dsh}/profiles/web/node_modules/@deepseek-ai/dsh-webhook-github/examples/github-review/cordis.yml"
```

アダプターは例を配布します。例は profile のインストールディレクトリから読み込まれるため、そのルールモジュールは profile にインストールされた peer を解決できます。profile 外の任意のディレクトリからルールを実行しないでください。カスタマイズするには、両方の例ファイルを `$DSH_HOME/profiles/web/github-review/` にコピーし、設定を編集し、`--patch` をその `cordis.yml` のコピーに向けます。`DSH_HOME` が未設定の場合は `~/.dsh` を使ってください。

リポジトリの例は `packages/experimental/webhook-github/examples/github-review/` にあります。CLI は webhook パッケージをインストールせず、Official plugin としてリストもしません。パッケージのインストールとオーバーレイの提供は別のステップです。

## 専用エンドポイントを公開する

メインの Web UI と `/api` はポート 3080 のままです。オーバーレイは分離された realm に2つ目の WebServer をマウントします。そこに登録されるのは `POST /github` のみで、他のすべてのパスは `404` を返します。

Caddy の設定でそのリスナーだけを公開できます:

```caddyfile
hooks.example.com {
  route {
    @github path /github
    reverse_proxy @github 127.0.0.1:3081
    respond 404
  }
}
```

GitHub を次のように設定します:

```text
Payload URL:  https://hooks.example.com/github
Content type: application/json
Secret:       DSH_GITHUB_WEBHOOK_SECRET value
Events:       Pull requests
Active:       yes
```

## ルールの動作

ルールはソース `primary-github`、リポジトリ `deepseek-ai/deepseek-harness`、イベント `pull_request`、アクション `ready_for_review` のみを受け付けます。正確な head SHA と選択された PR フィールドをレビュー prompt に渡し、JSON を信頼できないメタデータとラベル付けし、ファイル、ブランチ、PR、GitHub の変更を禁止します。

Session リクエストは `standard` agent preset と `read-only` permission preset を選択します。`workspacePath` は `WorkspaceRegistry.create()` を通じて正規化されるため、最初の一致する配信が Web Workspace を作成し（存在しない場合）、以降の配信はそれを再利用します。

HTTP レスポンスは意図的に Agent の結果より弱いものです。`202` は署名と JSON が受け入れられ、ルール呼び出しがメモリ内でスケジュールされたことを意味します。このルールが一致したことや、Session が作成されたことを意味しません。

## プログラムによる拡張

`run()` は通常の信頼された JavaScript です。デプロイは Session リクエストを返す前に内部ポリシーサービスへ問い合わせできます:

```js
const response = await fetch('https://policy.internal/pr-review', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ repository: payload.repository.full_name }),
  signal,
})
if (!response.ok || (await response.json()).automaticReview !== true) return null
```

リポジトリを異なるローカルパスにマップすることもできます:

```js
const workspacePath = {
  'deepseek-ai/deepseek-harness': '/path/to/deepseek-harness',
  'deepseek-harness/dsh-sdk': '/path/to/dsh-sdk',
}[payload.repository.full_name]
if (workspacePath === undefined) return null
```

## 配信セマンティクス

webhook runtime は配信や実行の状態を保存しません。繰り返し配信するとルールが実行され、別の Session が作成される可能性があります。クラッシュすると、prompt を受理していないルール呼び出しは失われます。prompt の受理後は、通常の Session ログ、永続化、Workspace、Agent ライフサイクルが作業を所有します。

webhook シークレットは受信 GitHub データのみを認証します。ルールコードにも、作成された Agent の送信 GitHub アクセスにも権限を与えません。ルールや Agent がその権限を必要とする場合は別途設定してください。
