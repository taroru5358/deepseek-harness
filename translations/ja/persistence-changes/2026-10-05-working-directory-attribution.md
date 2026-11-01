---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-05-working-directory-attribution

日本語 | [English](../../../docs/persistence-changes/2026-10-05-working-directory-attribution.md) | [中文](../../../docs/persistence-changes/2026-10-05-working-directory-attribution.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-05-working-directory-attribution.md](../../../docs/persistence-changes/2026-10-05-working-directory-attribution.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

プロデューサー所有の作業ディレクトリ通知の帰属を追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-05-working-directory-attribution
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-09-21-user-question-reply"
    after: "9a36727716d0e73f681f3b725e7f28f89417c68174c8af0ff2b1ef52b08326a0"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-09-21-user-question-reply"
    after: "ba5b10ea77d68f4a29439a9bd2a14f6a560acf55731749d57237757cc72fbc06"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-09-21-user-question-reply"
    after: "19fab83e64cf84ee8a6caf62c592e85c9b195bfd19e84a2667c6693a395681fd"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-09-21-user-question-reply"
    after: "2ace3b8059b82b6d9837fed518ca97a99eb37c4482df7aad0e71c85489428ade"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のレコードは有効なままである。追加された working-directory ソース種別は帰属のみである。宣言されたソース・ポリシーは未知の帰属を解釈せずに保持する。Session 形式は変わらない。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/session/working-directory/tests packages/shell/tool-bash/tests/tools.spec.ts packages/shell/tool-pwsh/tests/tools.spec.ts packages/subagent/subagent/tests/schedule-tools.spec.ts: 5 ファイル、207 件のテストに合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
