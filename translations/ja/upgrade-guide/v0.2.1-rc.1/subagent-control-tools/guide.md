---
kind: upgrade-guide
description: カスタムの subagent 構成は model-selection 設定を更新し、discovery を main エントリを通じて登録しなければならない。
---

# subagent のモデル選択と制御 tool を更新する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-tool-subagent` は `modelSelectionSettings` を `modelSelection` に置き換える。これは任意の子 route フィールドを制御する。discovery には tool scope ごとに1つのインスタンスで `modelDiscovery: true` が必要である。どちらのオプションもデフォルトは false である。このパッケージには `/list-models` plugin export はない。出荷される Web と Desktop の preset は spawn で両方のオプションを有効にし、fork の route フィールドは無効のままにする。カスタム fork tool は selection を有効にできる。Headless、SDK、ACP のデフォルトはそれぞれの tool セットを維持する。

Host の `/model-selection-settings` サービスは、委譲 tool がマウントされる前に作成された Session を含め、Agent 作成中に authorization を記録する。子 Session は policy を継承し、復元された Session はそれを保持し、後の tool インストールや設定編集は既存の Session を再サンプルしない。記録された policy がない場合、discovery と明示的な selection は現在の親 route のみを許可する。空の allowed-model リストで Host 設定を有効にすると、委譲 tool がなくても、新規のトップレベル Session がすべて拒否される。設定 UI はその組み合わせの保存を防ぐ。

`@deepseek-ai/dsh-tool-subagent-control` は `send_message`、`interrupt_agent`、`list_agents` を一緒に登録し、`tools`、`subagents`、`agents` を必要とする。その別個の `/list-agents` export は削除された。モデル discovery は、委譲 provider とは独立に、設定された委譲 tool インスタンスのライフタイムに従う。そのインスタンスをアンロードすると、他のインスタンスが残っていても discovery は削除される。`llm` が利用できない場合、discovery 呼び出しは失敗する。

## 移行

1. カスタム `cordis.yml`、パッチ、または直接の plugin 設定で、`modelSelectionSettings: true` を `modelSelection: true` に改名する。`list_subagent_models` を必要とする tool scope ごとに1つの委譲インスタンスで `modelDiscovery: true` を設定する。1つの scope に複数の discovery owner があると登録に失敗する。
2. カスタム構成でユーザー承認済みの代替 route が必要な場合は、Agent を作成する前に Host に `@deepseek-ai/dsh-tool-subagent/model-selection-settings` をマウントする。必要な `sessions` と `sessionProjections` サービスを提供する。その設定が有効な場合は、空でない allowed-model リストを提供する。直接 `apply(ctx, config)` を呼ぶ呼び出し元は、Session 引数を渡さなくなった。
3. `/list-agents` 行を、tool scope ごとに1つの `@deepseek-ai/dsh-tool-subagent-control` 行に置き換えるか、その行がすでに存在する場合は削除する。廃止された `tool-subagent-list-agents` 行を対象とするパッチを削除する。共有 control 行を無効にすると3つの tool すべてが無効になる。個々の tool を隠すには tool 制限を使う。
4. 新しい Session を作成し、要求した各 tool の定義が1つあることを確認する。preference が有効な場合は discovery が許可済み route を返し、そうでなければ親 route のみを返さなければならない。明示的な未承認の委譲は失敗しなければならない。discovery を所有するインスタンスをアンロードし、その discovery tool が消えることを確認する。
