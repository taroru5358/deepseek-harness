---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-22-action-card-status

日本語 | [English](../../../docs/persistence-changes/2026-10-22-action-card-status.md) | [中文](../../../docs/persistence-changes/2026-10-22-action-card-status.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-22-action-card-status.md](../../../docs/persistence-changes/2026-10-22-action-card-status.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

記録されるモデル・コンテキストにアクション・カードのステータス帰属を追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-22-action-card-status
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-10-13-user-rpc-principal"
    after: "b36fd842cd004b532ce96fd5d6f5be28e017eda043131aa3cf5e69edc436108e"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-10-13-user-rpc-principal"
    after: "5ef58f2a1bdc576c1f12fa1f4da1920ece020b13c0f991a578cddcbeb28bc5be"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-10-13-user-rpc-principal"
    after: "efa1a091017ce533e38b0ef3e14d11a5c102ff1793229b18dad719a4268d9adc"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-10-13-user-rpc-principal"
    after: "58b122df513aeaf7ddc5783dbb1fd844e2979bb273964e30ae351a8db6510019"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のメッセージとレコードは変わらない。action-card プラグインを持たないリーダーは、新しい action-cards ソース種別とその JSON メタデータを保持し、記録されたテキストを通常のコンテキストとして導出する。through シーケンスを使うのはプロデューサーだけであり、配信された決定を承認するために使う。これは権限を与えず、リプレイの解釈も変えない。

<a id="verification"></a>
## 検証

action-cards、creation-recommendations、client-ui-action-cards、client-ui-creation-recommendations の各テストで pnpm exec vitest run を実行: 120 件のテストに合格し、対象ソースを 100% カバーした。Web の creation-recommendations リプレイは、却下されたカードのステータス通知を次の回答の前に記録する。

<a id="dev-note"></a>
## 開発メモ

なし。
