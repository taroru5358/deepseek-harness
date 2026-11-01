---
kind: upgrade-guide
description: "Web 構成は Schedule 行を保持しなくなり、Automation tasks オプション bundle がそれらを挿入する。"
---

# Schedule は Automation tasks オプション bundle に移る

日本語 | [English](../../../../../docs/upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.md](../../../../../docs/upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.1.7-rc.2 では、`@deepseek-ai/dsh-web-app` が `time-context`、`schedule`、`ui-schedule` の行を `disabled: true` で保持していた。Plugins ページで Schedule を有効にするか、手動で有効にすると、`- id: schedule` と `disabled: false` のような id 対象の override が `$DSH_HOME/profiles/<name>/cordis.patch.yml` または `--patch` overlay に書き込まれた。

次のリリースでは Web 構成から3つの行が削除される。Plugins ページの Official グループに Automation tasks として表示される `@deepseek-ai/dsh-experimental-schedule-bundle` がそれらを挿入する。すべてのインストールで bundle はオフの状態で出荷される。

id で Schedule を有効にしていた profile は、アップグレード後にそれを失う。loader は `patch: entry schedule not found`（`time-context` と `ui-schedule` も同様）を警告し、`schedule_*` tool と Automation tasks ページが消え、保存済みリマインダーの配信が停止する。保存済み task と delivery record はディスクに残る。

## 移行

1. Plugins を開き、Official グループの Automation tasks を見つけて有効にする。スイッチは `$DSH_HOME/profiles/<name>/package.json` の `dsh.profile.bundles` に `@deepseek-ai/dsh-experimental-schedule-bundle` を追加する。手動で編集する profile では、そのエントリを自分で追加する。
2. `schedule`、`time-context`、`ui-schedule` の他のフィールド（`deliveryHistoryDays` など）を設定する既存の override は保持する。bundle が行を挿入すると再び適用される。`disabled: false` のみを設定する override は冗長なので削除できる。
3. 確認: `dsh web` を再起動し、起動ログに3つの id の `patch: entry ... not found` 警告がないこと、サイドバーに以前保存したリマインダー付きの Automation tasks が表示されることを確認する。
