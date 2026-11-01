---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-21-user-question-reply

日本語 | [English](../../../docs/persistence-changes/2026-09-21-user-question-reply.md) | [中文](../../../docs/persistence-changes/2026-09-21-user-question-reply.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-21-user-question-reply.md](../../../docs/persistence-changes/2026-09-21-user-question-reply.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

継続された ask_user_question 呼び出しへの遅れた回答のために、修飾された user-question-reply メッセージ・ソースを追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-21-user-question-reply
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-09-16-session-format-v4"
    after: "6178f1edcb23f361f2a6cb2c859b1c2187220d5695ece4a3400e0d92845a7178"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-09-16-session-format-v4"
    after: "186159f5f6f67a0b8cd095b8fe55bef42d4f25ca1a1c248f859867af2ece0467"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-09-16-session-format-v4"
    after: "ae84c5e493f94acc63cfb70389073ba616ed7ae7aa4fadde048bdcc64d49bb46"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-09-16-session-format-v4"
    after: "b83ed1b1cfffbd7bd5cca06ea72e44be57beb68b39e5a96660ce42a9e21aa411"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のログにはそのようなソースが含まれず、有効なままである。新しいソースは通常のユーザー・メッセージ上の修飾された帰属であり、dsh-user-questions を持たないリーダーはメッセージを保持し、その内容から履歴を導出する。userQuestions プロジェクションだけがこのソースを読み、名前付きの質問を閉じて回答を記録する。回答 RPC が唯一のプロデューサーであり、outcome answered を書き込む。Client パネルを閉じても返信は永続化されない。イベント型や Session ヘッダーの変更はない。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/interaction/user-questions/tests packages/interaction/tool-ask-user/tests: 81 件のテストに合格した。対象を絞ったプロジェクション、返信、ビュー、プロセス・グループのスイートが 70 件のテストに合格した。pnpm run typecheck に合格した。pnpm run doc-sync が、永続化の履歴と翻訳のペアリングを含む 42 個のゲートすべてに合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
