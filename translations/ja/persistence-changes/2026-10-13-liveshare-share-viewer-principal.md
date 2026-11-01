---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-13-liveshare-share-viewer-principal

日本語 | [English](../../../docs/persistence-changes/2026-10-13-liveshare-share-viewer-principal.md) | [中文](../../../docs/persistence-changes/2026-10-13-liveshare-share-viewer-principal.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-13-liveshare-share-viewer-principal.md](../../../docs/persistence-changes/2026-10-13-liveshare-share-viewer-principal.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

共有リーダーが送信するプロンプトの user-rpc メッセージ・ソースに、LiveShare の share-viewer プリンシパルを追加する。利用可能な場合は Relay が表明する表示名も含む。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-13-liveshare-share-viewer-principal
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-10-25-skill-catalog-incomplete"
    after: "cdf988172b589b230f2bce868489def1bb575eb0501076ac757a8f369ac5c64e"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-10-25-skill-catalog-incomplete"
    after: "70d2acdec1ebcfea56d02f7233ebb9ec0f7f494447fb952884c22c3afdc01d6f"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-10-25-skill-catalog-incomplete"
    after: "2e505098f9209e52eebd577f1a6f9d4e773ec4d78987aff28a7c786442b9f54c"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-10-25-skill-catalog-incomplete"
    after: "5bb231bc5afdb532a6898a20604e76bf0c1db438bfcdc17d4847dceaddc02451"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

share-viewer バリアントは、LiveShare の共有リーダーの Peer が送信するプロンプトにのみ書き込まれる。オーナーのプロンプトは引き続き principal を省略するため、そのソースと既存のすべての Session ログは変わらない。値は share id と、Relay が表明する provider、tenant、subject の id を持つ。Relay が表明する場合はオプショナルの表示名も持ち、これは帰属のみに使う。資格情報とアバターは決して永続化されず、認可は引き続き識別子の三つ組を使う。オプショナルのフィールドを知らないリーダーは、永続的な subject id を表示し続ける。LiveShare ホストは 2 か所でこれを消費する。`agent/pre-step` の受付チェックは、principal が share viewer である入ってくるすべてのメッセージについて Relay のグラントを再読し、そのグラントがもはや有効でない場合はそのステップを拒否する。同じステップは、実行中のターンがビューアー開始であったことをそのターンに限りメモリ内に記録し、これによりそのターンは共有ツール・サーフェスに限定され、オーナー開始のターンは Session ネイティブのツールのままになる。このマーカーは Session ログに追記されることはなく、次に受け入れられたステップから再計算される。LiveShare のないビルドでは、種別が 'user' のままであるためメッセージは通常のユーザー・メッセージとして保たれ、このパッケージが駆動しない principal 種別はそれを駆動する側に委ねられる。古いログの現在のリーダーは share-viewer プリンシパルを見つけず、それらのプロンプトを以前と同様に通常の入力として受け入れる。このフィールドはどのモデル・リクエストにも含まれない。SESSION_FORMAT_VERSION は変わらない。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/experimental/liveshare/tests/relay-target-admission.host.spec.ts packages/experimental/liveshare/tests/relay-pre-step-admission.host.spec.ts packages/experimental/liveshare/tests/share-host-listeners.spec.ts packages/experimental/liveshare packages/experimental/liveshare-protocol: 共有リーダーのプロンプトが agent/inbox/spliced に source.principal を永続化すること、復旧されたリーダー入力が `agent/pre-step` の受付チェックで拒否されること、受け入れられた入力はモデルに入ること、このパッケージが駆動しない principal はそのままにされることを確認する。オプショナルの displayName は帰属のために永続化され、空でない文字列としてのみ受け入れられ、欠落または不正な値は subject id にフォールバックする。35 件のテストに加え、2 パッケージの LiveShare とプロトコルのスイートが合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
