---
kind: upgrade-guide
description: "sdk-minimal profile は、session persistence が checkpoint を完了できない場合にモデルと tool の dispatch を停止するようになった。"
---

# SDK-minimal は永続的な checkpoint を必要とする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`sdk-minimal` profile は `session-checkpoint-policy` を有効にする。モデルリクエスト、トップレベル tool 実行、agent step の前に session 履歴の flush を待つ。以前は、JSONL 書き込みがバッファされている間もこれらの操作が進み、persistence エラーがバックグラウンド警告のままになることがあった。checkpoint の失敗は依存する操作を停止するようになった。初回使用時の file lock 初期化は最初のモデルリクエスト前に行われ、checkpoint 書き込みはリクエストレイテンシーに寄与する。これは Session 形式を変えず、強制終了が最後の未フラッシュ応答を保持することも保証しない。

## 移行

1. 選択した `dsh_home` を書き込み可能に保ち、リクエストの期限が storage 初期化と checkpoint 書き込みをカバーするようにする。shutdown timeout を長くするだけでは、リクエスト前の checkpoint には影響しない。
2. profile と呼び出しのパッチで `session-checkpoint-policy` の override を確認する。durability policy を受け取るには、`sessions` とともに有効なデフォルトを維持する。
3. 短い SDK turn を実行し、正常に閉じ、同じ home と session ID で `resume_session()` を使って復元を確認する。モデルや tool の作業を再試行する前に checkpoint の失敗を調査する。checkpoint が成功するまで操作は実行されない。
