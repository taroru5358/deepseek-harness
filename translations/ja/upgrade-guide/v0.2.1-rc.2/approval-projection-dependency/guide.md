---
kind: upgrade-guide
description: "カスタムの ApprovalService 構成は sessionProjections を提供し、サービス activation を待たなければならない。"
---

# Approval は Projection registry を必要とする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`ApprovalService` は `sessionProjections` を必要とする。registry が存在しないか pending の場合に event log を読む代わりに、activation 中に projection 登録を待つ。approval だけをマウントしていたカスタム構成は registry を追加しなければならない。出荷される profile はすでに提供している。

approval の結果、audit event、policy のデフォルト、同期ビジネスメソッドは既存の動作を維持する。Session log と checkpoint の形式は変わらない。private な登録ハンドルはプロセスローカルであり、checkpoint には policy override と turn 状態だけが含まれる。

## 移行

1. カスタム `cordis.yml` では、`@deepseek-ai/dsh-user-approval` の前に `@deepseek-ai/dsh-session-projection` をマウントする。出荷される profile に重複した registry を追加しない。
2. 直接のホスト構成では `ctx.plugin(ApprovalService)` を使い、`ctx.approval` を使う前に activation を待つ。`new ApprovalService(...)` でサービスを構築しない。これは `Service.init` を実行しない。

   ```ts
   import { Context } from '@deepseek-ai/cordis'
   import SessionStore from '@deepseek-ai/dsh-session'
   import SessionProjectionRegistry from '@deepseek-ai/dsh-session-projection'
   import ApprovalService from '@deepseek-ai/dsh-user-approval'

   const ctx = new Context()
   await ctx.plugin(SessionStore)
   await ctx.plugin(SessionProjectionRegistry)
   await ctx.plugin(ApprovalService)
   ```

3. 同期リーダーに履歴状態を公開する前に、ホストと registry が Session 回復を完了するようにする。[registry lifecycle](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md) に従う。approval getter や runtime hook の再生に履歴読み込みを追加しない。
4. activation が完了し、既存の approval テストが同じ結果と audit レコードを生成することを確認する。登録エラーは full-log fallback を選ぶのではなく activation を失敗させなければならない。
