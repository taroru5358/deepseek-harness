# Todo

日本語 | [English](../../../docs/subsystems/todo.md) | [中文](../../../docs/subsystems/todo.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/todo.md](../../../docs/subsystems/todo.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[`@deepseek-ai/dsh-tool-todo`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/todo/tool-todo/README.md)が所有する永続的な todo 語彙。モデル向けツールは 1 つの agent セッションのリスト全体を置き換え、パッケージはイベント宣言とリプレイ投影も所有する。ツールの振る舞いと設定は[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/todo/tool-todo/README.md)にある。

ソース: [`packages/todo/tool-todo/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/todo/tool-todo/src/types.ts)

## `TodoItem` — 1 つのリスト項目

```ts type-equiv
/**
 * One entry in an agent's todo list — the unit of the `todo/write`
 * whole-list snapshot declared by this package.
 *
 * Deliberately minimal: a human-readable `content` line and a three-state
 * `status`. No id, priority, or `activeForm` — the list is replaced wholesale
 * on every write (last-write-wins), so entries need no stable identity. The
 * three statuses describe the complete portable lifecycle needed by model and
 * UI consumers.
 */
interface TodoItem {
  /** What this task is — a short imperative line shown in the UI. */
  content: string
  /** Lifecycle state. `in_progress` marks a task being worked now; parallel work may mark several. */
  status: 'pending' | 'in_progress' | 'completed'
}
```

## 永続イベント

パッケージは `todo/write: { todos: TodoItem[] }` を `SessionEventMap` に宣言マージする。イベントはログ専用で、完全な置き換えリストを運ぶ。生成された[永続化カタログ](../../../docs/persistence-catalog.md#todowrite--log-only)がその宣言箇所を記録する。
