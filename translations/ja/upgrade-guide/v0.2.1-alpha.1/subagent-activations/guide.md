---
kind: upgrade-guide
description: "管理された subagent activation が foreground と Job ベースの委譲を置き換え、その設定キーを削除し、公開された in-process driver も置き換える。"
---

# 委譲を管理された subagent activation へ移行する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-tool-subagent` は `backgroundMode`、`enableRunInBackground`、およびモデル引数 `run_in_background` を削除する。すべての委譲は子を受け入れた後に `{ kind: 'activation', subagentId }` を返し、runtime は最終回答を親の完了通知で届ける。foreground の結果と `{ kind: 'background', jobId }` の結果は削除され、subagent は `job_output` や `job_kill` で収集・停止されなくなる。

公開パッケージ `@deepseek-ai/dsh-subagent-in-process-driver` は削除される。ローカル provider は `prepareContinuable()` を通じて作成入力を受け渡し、subagent サービスが実行を所有する。Spawn と Fork の子は、永続化と Session クエリが利用可能な場合、settlement 後も含めて後続メッセージを受け取れる。外部 provider は一度だけ実行し、フォローアップメッセージを受け取らない。

## 移行

1. `cordis.yml` の `tool-subagent` 行、profile パッチ、overlay から `backgroundMode` と `enableRunInBackground` を削除する。provider と残りの設定はそのままにする。例:

   ```yaml
   - name: '@deepseek-ai/dsh-tool-subagent'
     config:
       provider: spawn
       toolName: subagent
   ```

   ローカル tool の委譲には Session 永続化が必要である。[subagent サービスと provider](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent/README.md#use-this-package) と組み合わせて構成する。

2. tool 呼び出しから `run_in_background` を削除し、結果の consumer が `subagentId` を読むように更新する。foreground 出力や Job 結果の代わりに完了通知を収集する。現在の実行を停止するには `interrupt_agent` を使い、`send_message` はそれらの制御 tool が構成されている場合にローカルの子へ使う。

3. 結果を待つ必要があるホストコードでは、`ctx.subagents.start()` または `startContinuable()` を `startActivation()` と caller delivery に置き換える。構成済みサービスとライブな親がある場合:

   ```ts
   import type { Agent } from '@deepseek-ai/dsh-agent'
   import type SubagentRuntime from '@deepseek-ai/dsh-subagent'

   async function collect(subagents: SubagentRuntime, parent: Agent, signal: AbortSignal) {
     const activation = await subagents.startActivation({
       provider: 'spawn',
       label: 'Review changes',
       request: { parent, prompt: [{ type: 'text', text: 'Review these changes.' }] },
       signal,
       delivery: 'caller',
     })
     const result = await activation.result
     await activation.dispose()
     return result
   }
   ```

   caller delivery は親の完了通知を発行しない。`result` は cleanup 後に settlement し、`output`、`stopReason`、および任意の `structured` と `diagnostic` の値を含む。成功として扱う前に `stopReason` を確認する。startup signal は公開後、キャンセルの所有を停止する。receipt を保持し、`dispose()` を呼んでその activation だけをキャンセルする。cleanup の失敗は、捕捉された結果とは別に `dispose()` によって報告される。

4. driver の依存関係と import（`startInProcessRun` を含む）を削除する。既存の Spawn または Fork provider をマウントし、上記のようにサービスを呼び出す。カスタムのローカル provider は [provider API](../../../subsystems/subagent.md) を使って `prepareContinuable()` を実装する。代替のスタンドアロン driver はない。

   外部 provider は `start()` を保持するが、返される各 `SubagentRun.id` は runtime 内の親、provider、ローカル Session をまたいで一意でなければならない。親ローカルのカウンターではなく、実行ごとに新しい id を生成する。

   `dsh-subagent-dsh-sdk` については、別途設定された `dshBin` runtime を `session/wait` をサポートするよう更新する。ディレクトリ設定については、[working-directory の移行](../working-directory/guide.md) に従う。

5. 委譲が activation の結果 variant を返し、その完了通知に最終回答が含まれ、caller delivery がその通知を追加せずに結果を返すことを確認する。ローカルの子が完了後も一覧に残り、外部の子が継続できないことを確認する。
