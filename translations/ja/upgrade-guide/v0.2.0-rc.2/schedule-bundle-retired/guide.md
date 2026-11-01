---
kind: upgrade-guide
description: "Automation tasks オプション bundle は削除され、Web 構成が Schedule 自体をマウントする。"
---

# Automation tasks bundle の削除

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.2.0-rc.2 では、Plugins ページで Automation tasks を有効にすると、`$DSH_HOME/profiles/<name>/package.json` の `dsh.profile.bundles` に `@deepseek-ai/dsh-experimental-schedule-bundle` が追加された。その bundle は `time-context`、`schedule`、`ui-schedule` の行を挿入した。

次のリリースでは bundle が削除される。`@deepseek-ai/dsh-web-app` はすべての Web profile で `schedule` と `ui-schedule` をマウントし、`standard`、`cordis`、`ptc` の preset が clock 読み取りと4つの `schedule_*` tool を宣言する。`minimal` はどちらも宣言しない（[詳細](../../../subsystems/schedule.md)）。

profile をロードすると、その `dsh.profile.bundles` から `@deepseek-ai/dsh-experimental-schedule-bundle` が削除され、`package.json` が書き換えられる。他の manifest フィールドは保持される。保存済み task と delivery record はディスクに残り、引き続き使用される。

## 移行

1. 影響を受ける各 profile で `dsh` を一度起動する。手動編集は不要である。別の tool が書き込む profile ディレクトリは、`dsh.profile.bundles` からエントリ自体を削除しなければならない。
2. `cordis.patch.yml` または `--patch` overlay の `schedule` と `ui-schedule` への override は保持する。Web 構成がそれらの行を保持する。`time-context` へのトップレベル override は行に一致しなくなる。clock 行は preset に属するためである（[time-context](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/time-context/README.md)）。削除する。
3. 確認: `dsh.profile.bundles` が bundle を列挙しなくなり、Plugins ページに失敗した Automation tasks エントリが表示されず、サイドバーに以前保存したリマインダー付きの Automation tasks が表示される。
