---
kind: upgrade-guide
description: "persistent-terminal bundle の制御は Plugins ページから外れ、profile 設定が引き続き bundle を有効化・無効化する。"
---

# profile 設定を通じて persistent terminal を管理する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Web と Desktop は、すでに有効にしている profile も含め、Plugins リストや詳細ページに persistent-terminal bundle を表示しなくなった。パッケージと既存の activation はそのままである。minimal preset 自身の persistent shell はこの bundle とは独立している。

## 移行

1. 既存の選択を維持する場合、対応は不要である。変更する場合は `$DSH_HOME/profiles/<name>/package.json` を開く。デフォルトの home は `~/.dsh`、`<name>` は実行中の profile（`web` や `desktop` など）である。
2. 無効にするには `dsh.profile.bundles` から `@deepseek-ai/dsh-experimental-terminal-bundle` を削除し、有効にするには追加する。他のエントリと依存関係はそのままにする。
3. 同じ profile を再起動して選択を適用する。CLI 管理の profile では `dsh --profile <name> --dump-config` を実行する。`optional-persistent-terminals` グループは bundle が選択されている場合にのみ存在する。[profile リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md#profile-boot) を参照。
