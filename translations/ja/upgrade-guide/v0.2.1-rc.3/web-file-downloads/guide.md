---
kind: upgrade-guide
description: "カスタムの file-action メニューは、Desktop では save-as のみを、Web ではブラウザダウンロードを提供しなければならない。"
---

# Web でのファイルダウンロード

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Web の file-action サービスは、ブラウザのファイルピッカーを通じた save-as を提供しなくなった。Web では `canSaveAs` は常に false で、`save(source, 'saveAs')` は source を読む前に reject する。組み込みの Web ファイル・画像メニューは Download を提供し、Save as を省略する。Desktop は引き続き両方のアクションを提供し、成功した保存は不透明な file ID ではなく絶対的な宛先 `path` を返す。

## 移行

1. カスタム Client メニューは `ctx.fileActions.canSaveAs` が true の場合にのみ Save as を含めるべきである。
2. Web では `ctx.fileActions.save(source, 'download')` を呼ぶ。ブラウザが元の content と推奨ファイル名を処理する。`requested` の結果はダウンロードが開始されたことを意味し、完了したことを意味しない。保存されたローカルパスを報告しない。
3. `savedFile(url)` と `reveal(id)` の呼び出しを、完了した `saved` 結果に対するアクションに置き換え、その `path` を `session.openWorkspacePath` で使う。元のファイルの context-menu reveal は、保存先とは独立に保つ。
4. Web のファイル・画像メニューが、`showSaveFilePicker` をサポートするブラウザを含めて、Save as なしで Download を公開することを確認する。Desktop の画像メニューが両方のアクションを維持することを確認する。保存結果と feedback の所有権については [file-action リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-file-actions/README.md) を参照。
