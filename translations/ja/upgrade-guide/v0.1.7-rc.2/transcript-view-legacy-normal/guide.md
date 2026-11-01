---
kind: upgrade-guide
description: "レガシーな ui-chat.transcriptView 値 normal、および非 Desktop Web の未設定値は、Standard ではなく Detailed を表示するようになる。"
---

# レガシーな `normal` の作業詳細は Detailed として表示される

日本語 | [English](../../../../../docs/upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.md](../../../../../docs/upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Host 設定 `ui-chat.transcriptView` は Settings → General → Work details を選択する。v0.1.7-rc.2 では、client はレガシーな保存値 `normal` を `standard` として表示し、設定が欠落、`null`、または無効な場合はすべての client が `standard` を使っていた。

次のリリースから:

- 保存された `normal` は Desktop と Web で `detailed` として表示される。ディスク上の保存値は書き換えられない。
- 欠落、`null`、または無効な値は、非 Desktop Web（npm の `dsh web` インストール）で `detailed` として表示される。Desktop は引き続き `standard` を使う。
- 保存された `compact`、`standard`、`detailed`、`verbose` の値は変わらない。

古い表示に依存していたユーザーには、実行中の Turn が折りたたまれた要約ではなく展開された process-group 本文で表示される。

## 移行

1. 以前の表示を維持するには、Settings → General → Work details を開き、Standard を選ぶ。client は `standard` を `ui-chat.transcriptView` に保存し、以降のデフォルト変更では変更されない。
2. 確認: tool を呼び出す Turn を開始し、その process group がライブな task 詳細を伴う折りたたまれた要約を表示することを確認する。
