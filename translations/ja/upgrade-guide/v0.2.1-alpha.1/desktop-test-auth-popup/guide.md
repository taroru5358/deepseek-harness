---
kind: upgrade-guide
description: "Desktop の test build は、Feishu の update-policy authentication dialog を表示する前にローカルでのオプトインを必要とする。"
---

# ローカルで test update-policy authentication dialog を有効にする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

test build は以前、未認証の startup または手動の update-policy チェック後に Feishu ログインを提供していた。すべての test authentication dialog は現在デフォルトで無効で、手動チェックと mandatory-update refresh も含む。test gateway cookie は再起動を生き延びない。ログインが無効な場合、認証を要求する gateway は新しい mandatory-update decision を提供できない。設定を有効にし、再起動してサインインすると取得できる。認証失敗は、同じプロセスですでに知られている mandatory block を保持する。production policy リクエストと product account login は変わらない。

## 移行

1. 更新されたアプリケーションを一度起動し、存在しない場合は `app.getPath('userData')/desktop/settings.json` を作成する。既存ファイルは保持され、欠落フィールドは `false` にデフォルトする。プラットフォームごとのパスとエラー回復については[ローカル desktop 設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/README.md#local-desktop-settings)を参照。
2. Feishu ログインを許可するには、そのファイルで `updates.allowTestAuthPopupWindow` を `true` に設定し、Desktop を完全に終了して再起動する。未認証の test-policy 応答がログインを提供できるようになる。ログインダイアログが不要な場合はデフォルトを変更しない。
3. ダイアログを再び無効にするには、フィールドを `false` に設定して再起動する。startup と手動チェックは Feishu authentication dialog を表示してはならない。アップデートは引き続き policy を照会し、認証失敗は既知の mandatory block をクリアしない。
