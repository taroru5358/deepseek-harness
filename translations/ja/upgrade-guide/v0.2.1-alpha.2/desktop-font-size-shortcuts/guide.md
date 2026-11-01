---
kind: upgrade-guide
description: "Desktop のフォントサイズショートカットは、以前はカスタムショートカットに利用できたキー組み合わせを予約する。"
---

# Desktop のフォントサイズショートカットはカスタムのキー組み合わせを予約する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Desktop は `Cmd`/`Ctrl` + `=`（`+` のための `Shift` + `=` を含む）と `Cmd`/`Ctrl` + `-` を、テキスト、コード、terminal のフォントサイズをまとめて調整するために予約する。macOS では `Cmd`、Windows と Linux では `Ctrl` を使う。これらの組み合わせを使うカスタム binding は、重複する2キーの chord を含めて保存されたまま実行されなくなる。Web のショートカット動作は変わらない。

## 移行

1. Desktop で Settings → General → Keyboard shortcuts を開き、フォントサイズショートカットと競合するとしてマークされたカスタム binding を見つける。
2. そのアクションに競合しない組み合わせを記録するか、**Remove** を選んでキーボードショートカットを無効にする。アクションのデフォルトが競合しない場合は **Restore default** を利用できる。
3. 再割り当てしたアクションを試し、次に `Cmd`/`Ctrl` + `=` と `Cmd`/`Ctrl` + `-` を押す。3つすべてのフォントサイズが Settings で変更され、Desktop を再度開いた後も保存されていることを確認する。
