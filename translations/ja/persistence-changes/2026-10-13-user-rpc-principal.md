---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-13-user-rpc-principal

日本語 | [English](../../../docs/persistence-changes/2026-10-13-user-rpc-principal.md) | [中文](../../../docs/persistence-changes/2026-10-13-user-rpc-principal.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-13-user-rpc-principal.md](../../../docs/persistence-changes/2026-10-13-user-rpc-principal.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

オーナー以外の呼び出し元が送信するプロンプトのために、user-rpc メッセージ・ソースにオプショナルの principal を追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-13-user-rpc-principal
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-10-05-working-directory-attribution"
    after: "0755b7d1128319ba2e389619cbbf2c961ad648882b4e91da7f3e9582cc8a2034"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-10-05-working-directory-attribution"
    after: "97e0162f983c27456ec71269fb4976b1851588b8c1aeac5d071d43e11f95e03c"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-10-07-title-reasoning-effort"
    after: "79a4c87f8bfb4d6d05cd9952b99210d6a7db3b478c4a01d94b4c50602308c056"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-10-05-working-directory-attribution"
    after: "2ae8594a998c9862a597129e782277bcc518266712a703b5c3161bdf552d13b8"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

オーナーのプロンプトはこのフィールドを省略するため、そのソースと既存のすべての Session ログは変わらない。principal がない場合はオーナーを意味する。このフィールドより前のリーダーはそれを無視し、種別が 'user' のままであるためメッセージを通常のユーザー・メッセージとして扱う。古いログの現在のリーダーは principal を見つけず、すべてのメッセージをオーナーのものとして扱う。このフィールドはどのモデル・リクエストにも含まれない。プロバイダーのアダプターはモデルに見えるメッセージ内容だけを読む。SESSION_FORMAT_VERSION は変わらない。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/api/session-controller/tests/remote-access.host.spec.ts packages/subagent/subagent/tests/control.spec.ts packages/api/session-controller/tests/session-cold.host.spec.ts: オーナー以外のプロンプトが agent/inbox/spliced に source.principal を永続化し、オーナーと既存のタイムゾーンのソースはバイト単位で同一のままであることを確認した。すべてのテストに合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
