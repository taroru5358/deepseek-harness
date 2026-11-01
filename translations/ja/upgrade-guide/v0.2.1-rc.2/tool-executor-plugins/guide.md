---
kind: upgrade-guide
description: "tool の実行モードが dsh-tools の設定から明示的な executor plugin へ移る。"
---

# tool executor をマウントする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-tools` は tool 登録と executor 選択を維持するが、削除された `mode` と `maxParallelSubCalls` 設定フィールドをロード時に拒否する。native と PTC の実行には `@deepseek-ai/dsh-tool-executor-native` または `@deepseek-ai/dsh-tool-executor-ptc` が必要である。出荷される profile は executor をマウントするが、カスタム構成では1つ追加しなければならない。既存の `dsh-agent-tool-presentation` selector は引き続きサポートされるが deprecated である。起動時には有効な `tool-executor-native` と `tool-executor-ptc` 行が必要で、無効な executor 設定は起動を停止するようになった。Web、headless、SDK は `DSH_TOOLS_MODE` を読まなくなり、すべて base から native 実行を継承する。

## 移行

1. `dsh-tools` 行を維持し、その実行設定を削除する。意図したデプロイまたは preset scope に native または PTC executor を1つマウントする。出荷される profile パッチでは、`tools` 行ではなく executor 行を対象にする。
2. `maxParallelSubCalls` を PTC executor の設定に移し、互換性のある `ctx.ptcRuntime` provider をマウントする。executor はこのサービスを必須 injection として宣言する。runtime がない場合は起動が停止し、runtime をアンロードすると executor が撤回される。`agent-loop.maxParallelToolCalls` と保存済み override はデフォルト `10` のまま維持される。ループは現在の上限を各バッチに転送し、ライブ編集は次の tool グループではなく次のバッチに適用される。
3. カスタム executor は `describe` と `execute` を実装する。programmatic な呼び出し元は生の `calls` と `signal`、任意の `agent`、任意の `log: { turn, step }` を渡し、単一呼び出しの結果は `results` から読む。コミット済みモデル step の外では `log` を省略する。
4. PTC SDK renderer と `CodeRunFailedError` を `@deepseek-ai/dsh-tool-executor-ptc` から import する。選択した interaction mode で task を実行し、その tool 結果が Session に表示されることを確認する。
5. `ToolExecutionPipeline` と `executeToolBatch` を `@deepseek-ai/dsh-tools/pipeline` から import する。単一呼び出しを含む完了呼び出しはバッチを使う。カスタム scheduler は `prepare`、`dispatch`、`complete` を直接呼ぶ。tool fixture 用に `@deepseek-ai/dsh-tool-testkit` を開発依存関係として追加し、そのパッケージから import する。どちらも tool registry エントリからエクスポートされない。
6. 共有 dispatch event と `ptc-mode` の message-source 宣言を `@deepseek-ai/dsh-tool-executor-ptc/types` から import する。Client コードはその `/client` 再エクスポートを使う。Host policy 拡張は `PtcDispatchLog` と `tools/ptc-dispatch-log` のために `@deepseek-ai/dsh-tool-executor-ptc` から `import type` を使う。

7. 起動環境から `DSH_TOOLS_MODE` を削除する。PTC の場合、以下を `ptc.yml` として保存し、`--patch` で渡す。native 実行には overlay が不要である。同じパッチは `--profile web` と `--profile sdk` で動作する。TypeScript と Python の SDK client は `patches` を通じてそのパスを渡す。PTC モードでモデルが `run_code` だけを受け取ることを確認する。

```yaml
- id: tool-executor-native
  disabled: true

- id: tool-executor-ptc
  disabled: false
```

```bash
dsh --profile headless --patch ./ptc.yml "task"
```
