# Cordis チュートリアル

日本語 | [English](../../../docs/cordis-tutorial/index.md) | [中文](../../../docs/cordis-tutorial/index.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/index.md](../../../docs/cordis-tutorial/index.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Cordis は DeepSeek Harness の土台にあるプラグイン・フレームワークです。ツール、LLM アダプター、ファイル・アクセス、agent loop 自体まで、あらゆるケイパビリティが共有コンテキストにマウントされたプラグインである小さなランタイムです。このチュートリアルは Cordis を実践的に教えます。各章はこのリポジトリ内のスクラッチ・ディレクトリに構築する実行可能な例で、最後には実際の harness サービスに配線されたプラグインになります。

対象読者は agent 開発者です。深い TypeScript 経験は必要ありません。以下の [TypeScript ノート](#typescript-notes)が馴染みのない構文を説明し、各章が正確なコマンドと期待される出力を示します。

ウォークスルーではなく凝縮された概念リファレンスが必要なら、[Cordis 入門](../cordis-primer.md)を読んでください。網羅的な API リファレンスは、[サブシステムのページ](../subsystems/core.md)と [Cordis コア API](../../../docs/cordis-api/context.md) のページにある生成済みの `cordis-surface` 領域にあります。

harness 自体のプラグイン — 以下のランチャーではなく `cordis.yml` からロードされ、Web UI から駆動されるもの — を書く場合は、[最初の Harness プラグイン](../user/develop/basic/index.md)から始めてください。

## セットアップ

依存関係をインストールしたこのリポジトリのクローンが必要です。[開発ガイド](../development.md#setup-tutorial)に前提条件が列挙されています。このチュートリアルに API キーは不要です。すべての例はキーなしで実行できます。

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
```

章が作業するスクラッチ・ディレクトリを作成します。`tmp/` は gitignore されているため、そこに書くものはバージョン管理に触れません:

```sh
mkdir -p tmp/cordis-tutorial
cd tmp/cordis-tutorial
```

各章はこのディレクトリから同じコマンドを実行します:

```sh
node --import tsx ../../vendor/cordis/bin.js
```

その1ファイルのランチャー（[vendor/cordis/bin.js](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/cordis/bin.js) を参照）は、ルート `Context` を作成し、Loader プラグインをマウントし、カレント・ディレクトリから `./cordis.yml` をロードするよう指示します。それ以外のすべて — どのプラグインが存在するか、どのように設定されるか — は、これから書くその YAML ファイルから来ます。`--import tsx` フラグにより、Node は設定が指す TypeScript ファイルをビルド・ステップなしで実行できます。

## 章

1. [最初のプラグイン](01-first-plugin.md) — プラグインは関数であり、ローダーがマウントします。
2. [ライフサイクルと効果](02-lifecycle-and-effects.md) — Cordis が管理する登録は、プラグインのアンロード時に巻き戻されます。
3. [サービス](03-services.md) — `ctx` にケイパビリティを公開し、`inject` で依存します。
4. [イベント](04-events.md) — 型付きイベント、ブロードキャスト・ディスパッチ、waterfall のショートサーキット。
5. [設定](05-config.md) — `cordis.yml` からの検証済み設定。不正な入力では大きな音で失敗します。
6. [構成と HMR](06-composition-and-hmr.md) — プラグイン・ツリーとしての設定ファイル、ホットリロード、決してロードされないプラグインの診断。
7. [harness へ](07-into-the-harness.md) — 実際の harness サービスに対してモデル呼び出し可能なツールを登録します。

<a id="typescript-notes"></a>

## TypeScript ノート

例では、通常の現代 JavaScript に加えて3つの TypeScript 機能を使います:

- **型注釈**は実行時の振る舞いを変えずに値を記述します。`ctx: Context` は `ctx` が Cordis のコンテキスト API を持つことを示し、`who: string` はテキストを受け付け、`string[]` は文字列の配列を意味します。
- **`import type { Context } from '@deepseek-ai/cordis'`** は型情報だけを import します。実行時には消えるため、注釈のためだけに `Context` を必要とするプラグイン・ファイルは実行時依存を追加しません。
- **宣言マージ**（`declare module '@deepseek-ai/cordis' { ... }`）は、Cordis がすでに宣言しているインターフェースに自分のエントリーを追加します。たとえば新しい `ctx.greeter` プロパティやイベント名の型です。実行時の配線は生成しません。プラグインが別途サービスを提供するかイベントを emit します。第3章でパターン全体を示します。

第5章では、設定オブジェクトのフィールドを記述する `interface` と、スキーマがどのオブジェクト・フィールドを検証するかを示す `Schema<Config>` のようなジェネリック型も使います。これらの宣言は示されたとおりにコピーできます。周囲のテキストがそれぞれ何を結び付けるかを説明します。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
