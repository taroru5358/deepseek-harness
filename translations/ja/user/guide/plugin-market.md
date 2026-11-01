# plugin インデックス

日本語 | [English](../../../../docs/user/guide/plugin-market.md) | [中文](../../../../docs/user/guide/plugin-market.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/plugin-market.md](../../../../docs/user/guide/plugin-market.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

plugin インデックスは、インストール可能な DSH bundle を、タイトル、アイコン、作者、バージョンとともにリストする1つの JSON ファイルです。実験的な **Custom plugin indexes** bundle をオンにすると、Plugins ページの **Discover** タブでインデックスを閲覧し、確認後にリストされたバージョンをインストールできます。インデックスを公開するには、インデックスファイルを作成または生成し、アイコンファイルとともに任意の静的ホストから配信します。

## 目次

- [Discover を有効にする](#turn-on)
- [閲覧とインストール](#browse-and-install)
- [インデックスを公開する](#publish-an-index)
- [リファレンスジェネレーターでインデックスを生成する](#generate-an-index)
- [インデックス形式リファレンス](#index-format)
- [さらに調べる](#further-exploration)

<a id="turn-on"></a>
## Discover を有効にする

**Plugins** を開き、**Experimental plugins** の下にある **Custom plugin indexes** を見つけてオンにします。**Installed** の隣に **Discover** タブが現れ、Creator モードに読み取り専用の `plugin_market` tool が追加されます。bundle をオフにすると両方とも再び削除されます。

DSH はインデックスソースを同梱していません。**Discover** を選び、**Add index source** を選んで、インデックスファイルの URL を入力します。URL は `http:` または `https:` を使い、ユーザー名、パスワード、`#` フラグメントを含んではいけません。ネットワーク経路上の誰でも `http:` インデックスと、それがリストするインストール spec を変更できるため、管理下にないネットワーク経由で到達するインデックスには `https:` を使ってください。各ソースは現在の profile に属します。

<a id="browse-and-install"></a>
## 閲覧とインストール

ツールバー左側のインデックスメニューはソースを切り替え、**Manage index sources…** を開きます。そこでは各ソースが最後のエラーを表示します。検索は、あらゆる言語の名前、作者、キーワード、タイトル、説明、カテゴリ、ラベルに一致し、タイポも許容します。結果は別の並びを選ぶまでベストマッチ順です。**Labels** メニュー、カテゴリピル、**Sort** メニューは、インデックスが宣言するカテゴリ、ラベル、ソートキーを使い、**Publish date** は各 plugin の最新バージョンでソートします。

各カードは plugin の推奨バージョンを表示します。実行中の DSH と互換性がある、プレリリースでない最高バージョン、または plugin がプレリリースタグなしのバージョンをリストしていない場合は互換性のある最高プレリリースです。**Install** はそのバージョンのソース（npm パッケージとレジストリ、または Git リポジトリ、コミット、ディレクトリ）を含む確認を開きます。確認すると通常の plugin インストールが始まり、互換性とレジストリのチェックも含まれます。確認なしでインストールされるものはありません。

カードを選ぶとそのページが開きます。**Versions** リストはリストされたすべてのバージョンを互換性とともに表示し、DSH が受け入れる任意のバージョン（古いリリースやプレリリースを含む）の **Install** を提供します。**Installed**、**Update available**、または **Provided by DSH** を表示するカードは、代わりにインストール済み bundle を開きます。

インデックスはサードパーティのコンテンツです。インデックスは各 plugin のタイトル、説明、アイコン、作者、ソースを選びます。DSH はファイル形式をチェックしますが、それらの主張はチェックしません。信頼するソースのみを追加し、インストール前に確認画面でソースを確認してください。

<a id="publish-an-index"></a>
## インデックスを公開する

インデックスは UTF-8 JSON ファイルで、既定では最大 128 MiB、2xx ステータスでリダイレクトなしで提供されます。任意の静的ホスト、たとえば GitLab Pages、GitHub Pages、https 背後のオブジェクトストアに置きます。

アイコンはインラインまたは別ファイルにできます。インラインアイコンはインデックス内の base64 `data:` URL です。追加リクエストは不要ですが、インデックスが大きくなります。別ファイルのアイコンは URL です。相対アイコンはインデックス URL に対して解決されます。Web ページ内の画像がページに対して解決されるのと同じです。`https://market.example/dsh/index.json` のインデックスでは、`"icon": "icons/foo.svg"` は `https://market.example/dsh/icons/foo.svg` を読み込みます。したがって、インデックスと `icons/` ディレクトリを並べて公開すれば、ファイルを編集せずに任意のホストで動作します。別オリジン（スキーム、ホスト、ポート）の絶対アイコン URL は `https:` を使う必要があります。

これは npm plugin 1つと Git モノレポの plugin 1つを含む最小のインデックスです:

```json
{
  "format": 1,
  "title": { "en": "Team plugins", "zh": "团队插件" },
  "plugins": [
    {
      "name": "dsh-plugin-foo",
      "title": "Foo",
      "description": "Adds the foo tool",
      "icon": "icons/foo.svg",
      "author": "Jane Doe",
      "homepage": "https://github.com/jane/dsh-plugin-foo#readme",
      "repository": { "url": "https://github.com/jane/dsh-plugin-foo" },
      "versions": [
        { "version": "1.1.0", "publishTimestamp": 1790476800, "source": { "type": "npm" }, "engines": { "dsh": ">=0.2.0-rc.2" } },
        { "version": "1.0.0", "publishTimestamp": 1788000000, "source": { "type": "npm" } }
      ]
    },
    {
      "name": "@team/dsh-bar",
      "versions": [
        {
          "version": "0.3.0",
          "publishTimestamp": 1789000000,
          "source": {
            "type": "git",
            "url": "https://gitlab.example.com/team/dsh-plugins",
            "commit": "3f9c2e1d8a4b6c0e7f1a2b3c4d5e6f708192a3b4",
            "path": "packages/bar"
          }
        }
      ]
    }
  ]
}
```

以下のいずれかの規則に違反すると、DSH はインデックス全体を拒否し、**Manage index sources…** のソースの下に、問題のある値の JSON パスとともに理由を表示します。

<a id="generate-an-index"></a>
## リファレンスジェネレーターでインデックスを生成する

リポジトリには、プレーンなリストからインデックスを構築する[リファレンスジェネレーター](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/plugin-market-generator/README.md) が含まれています。1行に1つの npm パッケージ名または https Git リポジトリ URL を書きます。空行と `#` で始まる行は無視されます:

```text
@team/dsh-foo
https://gitlab.example.com/team/dsh-plugins
```

DSH チェックアウトから実行します:

```sh
pnpm --filter @deepseek-ai/dsh-plugin-market-generator start --input plugins.txt --output site/index.json --title "Team plugins"
```

npm パッケージの場合、インデックスは非推奨でもプライベートでもなく、`dsh.bundle` を宣言するすべてのバージョンをリストします。Git リポジトリの場合、既定ブランチ上で `package.json` が `"private": true` を設定していない bundle パッケージを見つけ、`vX.Y.Z`、`X.Y.Z`、`<name>@X.Y.Z` タグのバージョン、および既定ブランチ上の各パッケージのバージョンをリストします。タイトル、説明、アイコン、作者、ホームページは各 plugin の最高バージョンから取得します。パスは pnpm を実行するディレクトリからの相対です。

ジェネレーターはファイルをダウンロードして読むだけで、パッケージのインストールやコードの実行は行いません。リストの隣の `.plugin-market-cache` にキャッシュを保持するため、再実行時は新しいバージョンとコミットのみをダウンロードします。出力ディレクトリ、`index.json` とその `icons/` ディレクトリを、前述のように公開します。

<a id="index-format"></a>
## インデックス形式リファレンス

`LocalizedText` と記されたテキストフィールドは、文字列、または小文字の言語 id をキーとする文字列のオブジェクトです。たとえば `{ "en": "Foo", "zh": "甲" }` です。オブジェクトには `en` が必要です。DSH が知らないメンバーは無視されます。

| インデックスフィールド | 必須 | 意味 |
|---|---|---|
| `format` | はい | `1`。 |
| `title`, `description` | `title` のみ | インデックス名と説明。`LocalizedText`。 |
| `metadata.categories`, `metadata.labels` | いいえ | id を `{ "title": LocalizedText }` にマップするオブジェクト。 |
| `metadata.sortKeys` | いいえ | id を `{ "title": LocalizedText, "order": "asc" \| "desc" }` にマップするオブジェクト。`order` は初期方向です。 |
| `plugins` | はい | インデックスの既定順の plugin。 |

| plugin フィールド | 必須 | 意味 |
|---|---|---|
| `name` | はい | npm パッケージ名。インデックス内で一意。 |
| `title`, `description` | いいえ | `LocalizedText`。タイトルがない場合、カードは `name` を表示します。 |
| `icon` | いいえ | インライン `data:` URL（SVG、PNG、JPEG、または WebP、最大 32 KiB）または URL。前述のとおり。 |
| `author` | いいえ | 表示テキスト。 |
| `homepage` | いいえ | plugin についてのページの絶対 `https:` または `http:` URL。plugin ページがリンクします。 |
| `repository` | いいえ | `{ "url": string, "directory"?: string }`。ソースリポジトリの Web ページと、その中のパッケージディレクトリ。plugin ページは **Source code** としてリンクします。 |
| `metadata.category` | いいえ | 宣言されたカテゴリ id 1つ。 |
| `metadata.labels` | いいえ | 宣言されたラベル id。 |
| `metadata.sortKeys` | いいえ | 宣言されたソートキー id を数値にマップするオブジェクト。ダウンロード数や評価など。 |
| `versions` | はい | 少なくとも1つのバージョン。順序は任意。 |

| バージョンフィールド | 必須 | 意味 |
|---|---|---|
| `version` | はい | 正確な SemVer バージョン。plugin 内で一意。 |
| `publishTimestamp` | はい | Unix 秒での公開時刻。 |
| `source` | はい | `{ "type": "npm", "registry"?: url }`、または `{ "type": "git", "url": https-url, "commit": 40-hex-sha, "path"?: directory }`。 |
| `engines.dsh` | いいえ | このバージョンが動作する DSH バージョンの SemVer 範囲。ない場合は任意。 |

DSH は npm バージョンを `name@version` としてインストールし、`registry` がある場合は最初にそれを試します。Git バージョンは `git+<url>.git#<commit>` としてインストールし、サブディレクトリ内のパッケージには `&path:/<path>` を付けます。`engines.dsh` はパッケージ自身の `engines.dsh` と一致すべきで、DSH はインストール時に再チェックします。それがないパッケージは、代わりに `@deepseek-ai/dsh*` peer 依存関係に対してチェックされます。[形式ノート](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-10-02-plugin-market-index-format.md) がすべての規則を定義します。

<a id="further-exploration"></a>
## さらに調べる

- [Discover タブリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-plugin-market/README.md) はタブのコントロールと状態を説明します。
- [plugin インデックスサービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-market/README.md) は取得制限、キャッシュ、`plugin_market` tool を扱います。
- [インデックスバリデーター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/plugin-market-index/README.md) は、ジェネレーターが公開前にインデックスをチェックするために使えるライブラリです。

## 開発メモ

なし。
