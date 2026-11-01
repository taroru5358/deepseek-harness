---
kind: upgrade-guide
description: "Session のモデル選択は step の準備時に固定され、projection 型は所有パッケージへ移る。"
---

# Session のモデル選択は次の step に適用される

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

ACP のモデルと reasoning の変更は、現在の turn における tool continuation を含め、次のモデル step で有効になる。以前は ACP が完全な prompt turn に対して選択を固定していた。準備済みの step とその再試行は、捕捉された選択を保持する。画像の admission は admission 開始時に選択された route を確認する。後のオプション変更により、step 開始前に別の route が選択されることがある。

`AgentLoop` は永続的な `modelSelection` projection を所有し、各 Agent の `model` プロパティに `AgentModel` インスタンスを構築する。API の Session controller はそれを登録しなくなった。programmatic な resume は、`AgentOptions` にフォールバックする前に、未使用の選択を含む記録済みの選択を復元する。設定された Agent-loop の resume、headless adoption、SDK resume は、そのデプロイ route を記録する。省略された effort は、同じ route の場合にのみ保存済みの明示的な effort を保持する。

plugin は prompt assembly の前に provider、model、任意の明示的な reasoning effort を選択しなければならない。`agent/request` は deprecated である。既存の listener は各 main-loop リクエスト試行で引き続き実行され、他の制御を調整できるが、provider や model を変更したり、reasoning effort を追加・変更・クリアしたりはできない。effort がない場合は、各試行で adapter のデフォルトを通じて解決される。compaction などの補助呼び出しはこの event を dispatch しない。`installModelSelection` は switch-notice listener のみをインストールするようになり、その selection 引数と `ModelSelectionRef` は削除された。

client projection と Host state 型は `dsh-agent/types` に移り、API パッケージはそれらをエクスポートしなくなった。

## 移行

1. カスタム Cordis 構成では、`dsh-agent-loop` とともに `dsh-session-projection` をマウントする。ループが `agent.model` を構築するため、consumer は別個の model-selection サービスを必要としない。
2. 変更可能な参照と selection hook を、検証後の `agent.model.select(selection)` に置き換える。読み取りには `current()` を使う。switch notice のために Agent setup に `installModelSelection(agentCtx)` を残し、古い selection 引数を削除する。`agent/request` listener を追加しない。
3. 出力上限は `AgentOptions.maxTokens` を通じて設定する。その他のリクエストごとの制御 override には代替 API がない。その動作が依然として必要な場合は既存の listener を残す。
4. programmatic な resume 中に保存済み route を override するには、route が変わらず設定が effort を省略している場合にのみ、設定された selection を保存済みの明示的 effort とマージし、Agent setup callback で `agent.model.select(selection)` を呼ぶ。`agentOptions` だけを渡すと、selection またはリクエストのない Session に対するフォールバックのみを提供する。
5. 選択されている場合は明示的な effort を含め、捕捉した `Readonly<ModelSelection>` を `agent/pre-step.modelSelection` に渡す。`AssembleContext.modelSelection` は同じ selection を保持する。直接 `SessionReferenceResolver.prepare()` を呼ぶ場合は、route を4番目の引数として、任意の signal の前に渡す。
6. turn 全体で1つのモデルを必要とする ACP controller は、オプションを変更する前に active な prompt の完了を待たなければならない。turn 途中の変更が `model/selection` として記録され、次の step でそれが使われ、resume が未使用の選択を保持することを確認する。
7. `ModelSelectionProjection` を `@deepseek-ai/dsh-agent/types` から import し、API パッケージの alias を置き換える。`ModelSelection` は `dsh-agent/types` に残る。`ModelSelectionProjectionState` は driver を実装する場合にのみ `@deepseek-ai/dsh-agent` から import する。通常の読み取りは `agent.model.current()` を使う。effort 値は `dsh-llm/brand` の `ReasoningEffortId` を使う。
