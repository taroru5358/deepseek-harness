---
kind: upgrade-guide
description: "Auto review は入力 transport と既存履歴をまたいで user 由来の task 命令を認識する。"
---

# Auto review は user 由来の命令を認識する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

実験的な Auto review は、ブラウザの `rpcId` を必要とせずに、`source.kind: 'user'` のテキストを task 命令として認識する。これには Headless、SDK、ACP、user として送信するカスタム producer、既存の user 由来履歴が含まれる。明示的な命令は、その宣言された scope 内で medium-risk アクションを認可できる。子自身の作成 prompt と直接親からのメッセージは従属的な役割を保持し、継承され検証されていない作成 prompt は fact のままである。

## 移行

1. `createUserMessage` へのカスタム呼び出しを確認する。user source は user の task として送信された入力にのみ使う。background や注入された content には、producer が宣言する独自の `MessageSourceMap` エントリを与える。
2. Auto を resume する前に、medium-risk アクションを認可し得る既存の user 由来命令を見直す。次の task で古い scope を明示的に置き換える。
3. 注入された content が Auto のフィルタ済み履歴で `fact` または `constraint` として現れ、admission された user テキストが `user-instruction` として現れることを確認する。ファイル、tool 結果、checkpoint はアクションを認可できない。
