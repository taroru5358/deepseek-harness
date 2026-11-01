---
kind: upgrade-guide
description: "subagent の persona と toolFilter 設定は Agent Preset 構成に置き換えられる。"
---

# preset を通じた subagent 構成の設定

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`dsh-tool-subagent` は `persona` と `toolFilter` 設定を拒否する。新しい `SubagentStartRequest` 値と provider capability はどちらのフィールドも公開しなくなった。in-process spawn は `agentPreset` を通じて preset を選択し、その persona と tool はその preset の plugin から取得する。Fork は親の preset を継承し、persona を独立して変更したり tool を制限したりできない。model と reasoning-effort のオプションは変わらない。

出荷される `standard`、`ptc`、`cordis` の委譲 tool は `schedule_*` tool を隠さなくなった。委譲された呼び出し元は引き続き既存の実行拒否を受け取る。履歴上の子 Session descriptor は、resume 時に persona と tool 制限を保持する。

## 移行

1. profile パッチとカスタム preset の `dsh-tool-subagent` 行から `persona` と `toolFilter` を削除する。programmatic な呼び出し元と provider capability 宣言も同じフィールドを削除するよう更新する。
2. spawn では、`dsh-persona` と必要な tool plugin を含む Agent Preset を定義し、委譲 tool の `agentPreset` をその id に設定する。親モデルにいくつかの宣言済み preset を提供するには `allowedAgentPresets` を使う。preset は plugin 構成を選択する。子ごとの tool 名フィルタの代替はない。
3. fork では override を削除し、継承された構成を受け入れる。task が別の構成を必要とする場合は、preset 付きの spawn を使う。親の会話は継承しない。
4. 新しい Session を開始し、task を委譲する。子が選択した preset の persona と tool を使うことを確認する。既存の永続化された子 Session に手動編集は不要である。
