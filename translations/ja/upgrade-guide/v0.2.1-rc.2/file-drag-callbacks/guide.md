---
kind: upgrade-guide
description: "カスタムの画像 factory と Markdown 画像 delegate は file preparation callback を提供しなければならない。"
---

# 画像ビューに file preparation callback を提供する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

公開されている `conversation.images.content` Factory は `fileDrag` inject メンバーを必要とし、`MarkdownDelegate.fileImages` も同じ callback を必要とする。`ChatViewInjected` も `fileDrag` を得る。これらの値を構築するカスタム登録は、コンパイルが通る前に更新しなければならない。出荷される Web と Desktop の構成は `@deepseek-ai/dsh-client-ui-file-drag` を提供する。カスタム構成はこのサービスとその feedback slot を提供しなければならない。既存の画像 label フィールドは変わらない。

## 移行

1. カスタム client 構成に `@deepseek-ai/dsh-client-ui-file-drag` を、locale、slot renderer、`shell.overlay` を宣言する shell とともに含める。各 consumer plugin の `inject` リストで `fileDrag` を宣言する。
2. `conversation.images.content` の登録、`ChatViewInjected` の組み立て、`MarkdownDelegate.fileImages` の構築時に、`ctx.fileDrag.actions` を `fileDrag` として渡す。callback は Blob または URL の content を準備し、ローカライズされた失敗を報告する。コンポーネントはそれらを明示的に受け取る。任意の `resourceAddress(path)` は、認可された Desktop の source 再利用のために Session resource reference を提供する。
3. カスタム composer 実装は、独自の inject インターフェースを通じて `ctx.fileDrag.read` を受け取り、アプリケーションローカルの token を受け入れられる。`conversation.input.attachments` slot の公開 props は変わらない。出荷される attachment 実装のみがこの callback を必要とする。composer 領域で内部 drop を受け入れ、外部ファイルには通常の `dataTransfer.files` 処理を維持する。
4. カスタムパッケージを再ビルドし、composer への画像ドラッグを確認する。元のファイル名とバイト数を確認し、source に戻す drop が attachment を追加しないことを確認する。所有権とブラウザの制限については [file-drag plugin](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-file-drag/README.md) を参照。
