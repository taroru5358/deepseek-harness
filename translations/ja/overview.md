# DeepSeek Harness

日本語 | [English](../../README.md) | [中文](../../README.zh.md)

> これは非公式の日本語訳です。英語版（[README.md](../../README.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

DeepSeek Harness（`dsh`）は、[DeepSeek AI](https://deepseek.com) が開発したオープンソースの agent harness です。

**everything-is-a-plugin** アーキテクチャを採用し、[Cordis](https://github.com/cordiverse/cordis) 上に構築されています。Cordis の設計は、論文 [_A Programming Paradigm for Spatiotemporal Composability_](https://arxiv.org/abs/2608.25512) で説明されています。

ドキュメント: [https://deepseek-harness.github.io/deepseek-harness/](https://deepseek-harness.github.io/deepseek-harness/)

日本語ドキュメント: [translations/ja/index.md](index.md)

## 開発者プレビュー

DeepSeek Harness は _開発者プレビュー_ の段階にあり、急速に改善されています。**互換性を壊す変更が入る可能性があります。**

プロジェクトを実行する前に、[安全性に関する注意](../../SAFETY.md)を確認してください。

## 実行

### `npm` から実行する

`Node.js` をインストールして、次を実行します。

```sh
npx @deepseek-ai/dsh web
```

このコマンドは既定で `http://127.0.0.1:3080` に Web UI を起動し、ローカル起動の場合は既定のブラウザで開きます。SSH 経由の起動では、SSH クライアントまたはエディタがローカルの転送先アドレスを管理するため、ホスト URL の表示だけを行います。ブラウザを開かずにサーバーを起動するには `--no-open` を指定します。詳しくは [Web UI ガイド](../../docs/user/guide/index.md)を参照してください。

### ソースから実行する

リポジトリのチェックアウトから実行するには、次を実行します。

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

`pnpm run build` はリポジトリの成果物を準備します。`pnpm dsh web` は、再ビルドせずにその成果物を使います。

## コミュニティとサポート

- フィードバックやバグ報告は [GitHub Discussions](https://github.com/deepseek-ai/deepseek-harness/discussions) から送ってください。
- プラグインのリポジトリには [`dsh-plugin`](https://github.com/topics/dsh-plugin) トピックを付けると、見つけてもらいやすくなります。
- <a href="https://discord.gg/4MrtZUhpxg">DeepSeek Harness Discord コミュニティ</a>に参加してください。

## コントリビューション

[CONTRIBUTING.md](../../CONTRIBUTING.md) を参照してください。

## 開発

まず [開発ガイド](../../docs/development.md) と [アーキテクチャドキュメント](architecture.md) を読んでください。

`pnpm run dev:web` は、1つのターミナルでクライアントバンドルをビルド・配信し、ソース編集時に再ビルドします。`make help` は、Web と Desktop に対応する Make ターゲットを一覧表示します。アプリケーションコマンドの完全な表は、開発ガイドの該当セクションにあります。

エージェント向けの指示は [AGENTS.md](../../AGENTS.md) に従ってください。

## 引用

```bibtex
@misc{deepseek-harness2026,
  title={DeepSeek Harness: Everything is a Plugin},
  author={DeepSeek-AI},
  year={2026},
  publisher={GitHub},
  howpublished={\url{https://github.com/deepseek-ai/deepseek-harness}},
}
```

## ライセンス

[MIT](../../LICENSE)

サードパーティ依存関係とそのライセンスは [THIRD_PARTY_NOTICES.md](../../THIRD_PARTY_NOTICES.md) に記載されています。
