---
kind: upgrade-guide
description: "SSH と Node PTC の構成は明示的な launch mode を使い、SSH client は一致する protocol-2 helper を必要とする。"
---

# SSH helper と PTC の起動設定

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

カスタム SSH 構成は `cordis.yml` とインストール済み helper を一緒に更新しなければならない。`dsh-ssh` はトップレベルの `node`、`bootstrapPath`、`bootstrapHash` を `launch` に置き換え、SSH protocol 2 を必要とする。`dsh-ptc-runtime-node` は `nodeExecutable` と `bootstrapPath` を `launch` に置き換える。connection の `nodeExecutable` と `bootstrapPath` getter は `ptcLaunch` に置き換えられる。

## 移行

1. script helper では、`node`、`bootstrapPath`、`bootstrapHash` を `launch: { kind: node-script }` の下に移す。`helper`、`helperHash`、`host`、`workspace` は既存の位置に保つ。一致する helper script をインストールして `helperHash` を更新する。設定されている場合は bootstrap とその hash を一緒に更新する。
2. packaged helper では、完全なアーカイブを展開し、`launch: { kind: executable }`、`helper` を実行ファイルの絶対リモートパス、`helperHash` を `manifest.json` の digest に設定する。
3. リモート PTC provider は、構成の既存の JavaScript 設定で `launch: ctx.ssh.ptcLaunch` を使って設定する。明示的に設定されたローカル Node worker には `launch: { kind: 'node-script', executable: previousNodeExecutable, bootstrapPath: previousBootstrapPath }` を使う。ローカル起動設定を省略した場合は、引き続き現在のプロセスの runtime が選択される。
4. 設定した profile を起動し、リモートのファイル読み取り、プロセスコマンド、PTC 呼び出しを実行する。protocol や digest の不一致は、インストール済み helper とその設定を一緒に更新する必要がある。[SSH デプロイの要件](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ssh/ssh/README.md#use-this-package) を参照。
