---
kind: upgrade-guide
description: "コマンドライン provider を無効にする headless 構成は、sandbox policy の設定を直接提供しなければならない。"
---

# headless startup を置き換える場合の policy 設定

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

headless bundle は sandbox policy を activate する前に `headlessStartup` を通じて `--add-dir` を読む。`headless-startup` を無効にする overlay は、`sandbox-policy` からその依存関係も削除しなければならない。そうしないと、policy とそれに依存する tool が pending のままになる。通常の `dsh headless` 呼び出しには設定変更は不要である。

## 移行

1. `headless-startup` を無効にする overlay で、policy を直接提供する。この置き換え設定に既存の policy 設定を保持する:

   ```yaml
   - id: sandbox-policy
     inject: []
     config:
       mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'
       workspaceRoot: !!js process.cwd()
   ```

2. overlay がコマンドライン startup を維持しつつ `headless-runner` または `sandbox-policy` を置き換える場合は、[headless bundle](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/cordis.patch.yml) からそれらの `headlessStartup` binding を保持する。
3. カスタム profile を通じて小さな task を起動し、その tool が activate することを確認する。startup が有効なままの場合は `--add-dir` を渡し、選択したディレクトリへの書き込みが成功することを確認する。
