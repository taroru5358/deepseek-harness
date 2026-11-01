---
kind: upgrade-guide
description: "Forge の Git クエリは Session 権限を使い、設定された diff helper を無効にする。"
---
# Forge の Git クエリ権限と content conversion

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Forge の Git クエリは `read-only` を強制する代わりに Session の sandbox mode を使う。`danger-full-access` は sandbox バックエンドなしで動作し、`workspace-write` は設定された root 内での書き込みを許可する。制限付きモードは、バックエンドが利用できない場合に引き続き実行を拒否する。

Git 2.31 以降では、ローカルの diff と統計がコマンドごとに列挙された clean/process filter を無効にする。Git 組み込みの改行、ident、encoding 変換は引き続き適用される。実体化された LFS ファイルや他のフィルタ済みファイルは、通常の Git が clean と見なす場合でも変更ありとして表示されることがある。パネルは LFS のダウンロードを要求しない。古い Git や同時に追加された filter は、Session 権限の下で helper を実行する可能性がある。submodule の commit 変更は引き続き表示されるが、内部の未コミット変更は省略される。

filter 設定の読み取り失敗（切り詰められた出力を含む）は、比較の前に `forgePanel/filter-config-failed` を報告し、再試行を提供する。これらは `forgePanel/diff-too-large` とは別である。

## 移行

1. Git クエリに OS が強制する書き込み保護が必要な場合は、Session を `read-only` に保つ。既存の permission picker を使って意図した Session mode を選択する。設定キーは変わらない。
2. content conversion や submodule working tree の検査が必要なリポジトリでは、意図した Session 権限の下で明示的な Git コマンドを使ってそれらの変更を調べる。パネルのローカル比較は、サポートされる Git バージョンで clean/process filter をバイパスする。
3. sandbox バックエンドのない host では、Full access を選択して Forge パネルを更新する。checkout と通常のローカル変更がロードされるはずである。制限付きモードに切り替えると、引き続き利用できないバックエンドを報告しなければならない。
