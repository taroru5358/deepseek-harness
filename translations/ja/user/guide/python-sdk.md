# Python SDK を使い始める

日本語 | [English](../../../../docs/user/guide/python-sdk.md) | [中文](../../../../docs/user/guide/python-sdk.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/python-sdk.md](../../../../docs/user/guide/python-sdk.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このチュートリアルでは、公開されている Python SDK をインストールし、同梱のスタンドアロン最小 profile を実行し、同じ `dsh` profile を自分のプログラムからカスタマイズする方法を示します。

## 前提条件

- Python 3.10 以降
- Git
- Linux x64、Linux arm64、arm64 上の macOS 14 以降、または Windows x64
- DeepSeek 互換の API エンドポイントとクレデンシャル
- 分離された workspace と分離された Harness home

## SDK をインストールする

<div>
<a id="linux-and-macos"></a>
<a id="windows-powershell"></a>
</div>

::: code-group

```sh [Linux/macOS]
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
python -m venv .venv
. .venv/bin/activate
python -m pip install deepseek-harness-sdk
```

```powershell [Windows PowerShell]
git clone https://github.com/deepseek-ai/deepseek-harness.git
Set-Location deepseek-harness
py -3.10 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install deepseek-harness-sdk
```

:::

インストールには対応するネイティブ runtime wheel と `dsh` コマンドが含まれます。通常の SDK 実行にシステムの Node.js は不要です。成果物をビルドするリポジトリのコントリビューターは [Python コントリビューターワークフロー](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/development.md) を使ってください。

## 同梱の例を実行する

クレデンシャルをエクスポートし、必要に応じて互換プロキシエンドポイントもエクスポートします:

<div>
<a id="linux-and-macos-1"></a>
<a id="windows-powershell-1"></a>
</div>

::: code-group

```sh [Linux/macOS]
export DEEPSEEK_API_KEY=sk-your-key-here
# export DEEPSEEK_BASE_URL=http://127.0.0.1:8000/v1
```

```powershell [Windows PowerShell]
$env:DEEPSEEK_API_KEY = "sk-your-key-here"
# $env:DEEPSEEK_BASE_URL = "http://127.0.0.1:8000/v1"
```

:::

明示的な workspace と home パスを指定して、1つのタスクを実行します:

<div>
<a id="linux-and-macos-2"></a>
<a id="windows-powershell-2"></a>
</div>

::: code-group

```sh [Linux/macOS]
python python/sdk/examples/minimal.py \
  --workspace /absolute/path/to/disposable-workspace \
  --dsh-home /absolute/path/to/example-dsh-home \
  --session-id example-001 \
  "Inspect the repository and fix the failing tests."
```

```powershell [Windows PowerShell]
python python/sdk/examples/minimal.py `
  --workspace C:\work\disposable-workspace `
  --dsh-home C:\work\example-dsh-home `
  --session-id example-001 `
  "Inspect the repository and fix the failing tests."
```

:::

スクリプトは、選択した profile が提供する最終的な assistant 応答またはコマンド結果を出力します。選択した home には、生成された `sdk-minimal` profile、インストール済み plugin、および `sessions/` 下の非圧縮 JSONL session ログが置かれます。この例と SDK は `~/.dsh` を暗黙に読むことはありません。

## プログラムで SDK を使う

```python
from pathlib import Path

from deepseek_harness import DeepSeekHarness

workspace = Path("/absolute/path/to/disposable-workspace").resolve()
dsh_home = Path("/absolute/path/to/example-dsh-home").resolve()
with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    max_tokens=49_152,
    cwd=str(workspace),
    dsh_home=str(dsh_home),
    profile="sdk-minimal",
) as harness:
    result = harness.run(
        "Inspect the repository and fix the failing tests.",
        session_id="example-001",
    )

print(result.final_response)
```

SDK は同梱の `dsh --profile sdk-minimal` プロセスを遅延起動し、context manager を終了するまで再利用します。profile、その永続 patch、home patch、および順序付きの `patches` タプルがアプリケーション設定を構成します。別個の Python runtime bin や完全設定オプションはありません。

## plugin をインストールまたは定義する

この home に永続化すべき依存関係と bundle レイヤーには `dsh plugin` を使います:

<div>
<a id="linux-and-macos-3"></a>
<a id="windows-powershell-3"></a>
</div>

::: code-group

```sh [Linux/macOS]
export DSH_HOME=/absolute/path/to/example-dsh-home
dsh --profile sdk-minimal --dump-default-config >/dev/null
dsh plugin --profile sdk-minimal add file:/absolute/path/to/my-plugin-bundle
```

```powershell [Windows PowerShell]
$env:DSH_HOME = "C:\work\example-dsh-home"
dsh --profile sdk-minimal --dump-default-config | Out-Null
dsh plugin --profile sdk-minimal add file:C:/work/my-plugin-bundle
```

:::

最初のコマンドは同梱のスタンドアロン profile を初期化します。2番目はパッケージ管理を `pnpm` に転送し、`dsh.bundle` レイヤーをエクスポートするインストール済みパッケージを記録します。`pnpm` はこの管理コマンドにのみインストールしてください。インストール済み SDK の起動には不要です。永続的な行の変更には `$DSH_HOME/profiles/sdk-minimal/cordis.patch.yml` を編集し、起動ごとの変更には Python から patch ファイルを渡します。

別の `profile` も、`@deepseek-ai/dsh-sdk-app` または別の JSON-RPC サーバー行を含んでいれば有効です。サーバー行の欠落、未解決の plugin、無効な patch は、別の構成にフォールバックするのではなく起動時に失敗します。

<a id="opt-in-to-str_replace_editor"></a>
### `str_replace_editor` をオプトインする

同梱 runtime には `str_replace_editor` が含まれますが、`sdk-minimal` は既定の Cordis ツリーからそれを省略しています。使うには、この設定を `editor.patch.yml` として保存します。`insert` は既存の filesystem provider を使ってエディターを追加します:

```yaml
- insert:
    - id: tool-str-replace-editor
      name: '@deepseek-ai/dsh-tool-str-replace-editor'
```

`DeepSeekHarness(profile="sdk-minimal", ...)` を構築するときに `patches=("/absolute/path/to/editor.patch.yml",)` を渡すか、永続設定として patch を `$DSH_HOME/profiles/sdk-minimal/cordis.patch.yml` に置きます。次回の runtime 起動では、モデルリクエストに既定の tool と並んで `str_replace_editor` が含まれます。ローカル filesystem provider は相対パスに runtime の作業ディレクトリを使います。最小 shell と同様に、そのディレクトリへのアクセスを制限しません。同じ patch は標準の `sdk` profile でも動作します。

## 最小 profile を理解する

| プロパティ | 値 |
|---|---|
| システムプロンプト | `DSH_SYSTEM_PROMPT`、なければ `You are a helpful software engineer assistant.` |
| `minimal.py` のモデル | `--model`、次に `DSH_MODEL`、次に `deepseek-v4-flash` |
| モデル向け tool | Linux/macOS では永続 `bash`、Windows では `pwsh`、`read_image`、`working_directory` |
| shell タイムアウト | 300 秒 |
| オプションの runtime コンテキスト | なし |
| コンテキストの compaction | 自動。追加のモデルリクエストで古い履歴を要約 |
| session の永続化 | `<dsh_home>/sessions` 下の非圧縮 JSONL |

この profile の唯一の bundle は、空のルートに完全なツリーを挿入し、`dsh-base` を含みません。そのため、後の base profile の tool が暗黙に現れることはありません。SDK プロトコル、環境変数で設定される DeepSeek アダプター1つ、ローカル実行、永続化を含みますが、テキストファイルツール、設定、管理クレデンシャル、OTel テレメトリー、Web ツール、サブエージェント、ローカル指示の検出は含みません。[DeepSeek session ログコントリビューター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/README.md) は、既定で DeepSeek リクエストとともに未受理のログイベントを、リクエストあたり最大 `maxBytes`（8 MiB）までアップロードします。無効にするには profile patch で `session-log-deepseek.enabled: false` を設定します。これは `danger-full-access` を固定するため、プラットフォームが選択する永続 shell は runtime から見える任意のパスを変更できます。使い捨てのチェックアウトまたはコンテナを使ってください。

画像読み取りには画像対応のルートが必要です。自動 compaction は完全な Session ログを保持します。モデルとストレージへの影響は [bundle リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) を参照してください。この `sdk-minimal` 構成では、サポートされないスラッシュ入力は通常の prompt テキストのままです。

インストール済み wheel は依然として完全な `web` profile とフロントエンドアセットを同梱します。Python SDK のデプロイでブラウザアプリケーションも必要な場合は、明示的な `DSH_HOME` に対して `dsh web` を実行します。`web` は別個の CLI アプリケーションであり、Python SDK クライアントにサービスを提供できません。

profile、plugin、クレデンシャル、設定、session を分離する必要がある場合は、新しい home を使ってください。独立した作業には新しい session id を使い、同じ永続的な会話と session 所有リソースを継続する場合にのみ harness、home、id を再利用します。

[bundle リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) が正確なツリーを管理し、[example リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/sdk/examples/README.md) が実行可能なプログラムを管理します。[Python SDK リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/sdk/README.md) はライフサイクル、結果、通知、低レベル動作を扱い、[dsh CLI リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md) は profile のレイヤー化を扱います。
