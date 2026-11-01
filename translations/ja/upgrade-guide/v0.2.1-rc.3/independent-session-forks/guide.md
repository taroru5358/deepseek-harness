---
kind: upgrade-guide
description: "通常の Session fork は source 関係の記録を停止し、subagent として表示されなくなる。"
---

# 通常の fork は独立した Session になる

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

通常の fork は選択した履歴をコピーし、独立して継続する。`SessionHeader.parentSession` と Client の `parentSessionId` を省略し、代わりの source フィールドも書き込まれない。fork バックエンドの subagent を含む subagent は、直接の作成者を保持する。既存の log は記録済みメタデータを保持する。

SDK の `subagent.started` は通常の fork を除外する。親 id を持つ履歴上の fork も含まれる。ACP の list/resume は通常の fork を許可し、headless adoption も cwd と preset のチェックが通れば受け入れる。Client のリストはそれらをインデントせず、新しく発見された fork をその source の前に挿入しなくなる。保存された手動位置はそのまま残る。source ベースの trace クエリ、parent filter、telemetry、remote log は、新しい通常 fork の source を復元できない。workspace binding、任意の title increment、継承履歴、seed boundary は引き続き利用可能である。

Session 形式は V4 のままである。古い V4 reader は、継承された delivery acknowledgement を含む parentless fork を、`current-generation delivery marker names the wrong Session` という corrupt log として拒否する場合がある。テスト済みの reader は handle を渡す前、またはファイルを切り詰める前に拒否する。アップグレード固有の診断は提供しない。[互換性の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-10-23-independent-session-forks.md) を参照。

## 移行

1. 新しく作成された fork を resume する前に、Session ディレクトリを共有するすべての reader と writer をアップグレードする。古い reader の拒否を回避するために古いヘッダーを編集したり、delivery record を削除したりしない。
2. 任意の `parentSession` を subagent として扱う consumer のチェックを `origin === 'subagent'` に置き換える。継続には引き続き Session 自身の event range からの有効な descriptor が必要である。継承された descriptor はその identity を確立しない。
3. 永続的な source trace を必要とする通常 fork のワークフローを削除する。workspace 選択のような source 依存のアクションは、fork 操作内に保つ。
4. 通常の fork に parent フィールドがなく、継承された delivery record とともに再起動を生き延び、`subagent.started` を生成しないことを確認する。実際の subagent が作成者と通知を保持することを確認する。
