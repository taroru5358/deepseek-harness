# プラン・モード

日本語 | [English](../../../docs/subsystems/plan.md) | [中文](../../../docs/subsystems/plan.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/plan.md](../../../docs/subsystems/plan.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

プラン・モードは、[dsh-plan-mode](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/plan/plan-mode)（`ctx.planMode`、`PlanModeController`）が所有する agent ごとの記録されるコラボレーション状態である。アクティブな間、デプロイが所有するガイダンス・セクションが各モデル要求に含まれる。プラン・モードは**ソフトなガイダンス**である。[サンドボックス・モード](sandbox.md)と[承認ポリシー](approval.md)は独立して制限を強制し、どちらもプラン状態を読んだり書いたりしないため、デプロイはそれらを別々に設定する。このパッケージは任意であり、agent ループはそれに依存しない。`plan:policy` プロンプト・セクションを提供し、`exit_plan_mode` ツールと `/plan` コマンドを登録する。根拠は[設計リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/README.md)が所有し、モデル体験と制限の詳細は[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/README.md)が所有する。

ソース: [`packages/plan/plan-mode/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/src/index.ts)

## 記録される状態と回復

`plan/mode`（`{ active: boolean }`）はログ専用で値全体を置き換える[セッション・イベント](session.md)であり、永続的でリプレイ可能で、モデルのトランスクリプトには決して入らない。`plan` ユニットは、コミットされたモード、コマンドの決着、最新の要求ヘッダーで記録されたモードを畳み込む。`ctx.planMode` はレジストリを必要とし、アクティブ化中に `plan` と `turnBoundary` の登録を待ち、その後は保持されたハンドルを通じて読む。Clients は `{ active, pending }` だけを受け取る。再開、フォーク、圧縮はどちらもログから回復する。完全なイベント宣言は[永続化ログ・イベント・カタログ](../../../docs/persistence-catalog.md)にある。

## 保留中の選択とステップ前の追加

すべてのセッション・イベントはターンで囲まれるため、ユーザーの選択は、次に受け入れられたターン内の pre-step が、要求の導出前にそれを追加するまで保留のまま残る。それがどのターンで起こっても同じである。選択は継続を強制しないので、ターンの最後に受け入れられた pre-step の後に行われた選択は、後のターンで追加される。`set(agent, active)` は保留中の選択を記録し（ターゲットが記録済みまたはすでに保留中の状態と等しい場合は no-op）、`get(agent)` は `{ active: boolean; pending?: boolean }` を返す。現在のステップを組み立てるために使われた記録済みの状態と、追加を待っている選択された状態である。

agent が実行中の唯一の追加ポイントは、先頭に追加された `agent/pre-step` リスナーである。これは提案されたすべての要求ステップを観測し（ターン 1 ステップ 1 と要求回復の再試行を含む）、下流のリスナーを先に呼び、それらがステップを受け入れた後にのみ追加する。プロンプトのアドミッションはターンの前に起こり、`plan/mode` を追加できないので、プロンプトで行われた選択は、それが開始するターンの最初に受け入れられたターン内の pre-step によって追加される。追加の失敗はターンをブロックできず、選択は後の受け入れられたターン内の pre-step のために保留のまま残る。追加されたユーザーの選択は、最後に記録された要求ヘッダーがもう一方の状態を記述していた場合に限り、プラグイン由来の `user/message` 通知を 1 つ記録するので、モデルは自身のコンテキストが変わった正確な時点を知り、冗長に知らされることはない。ターンの最後に受け入れられた pre-step の後に行われた選択はプロセス・ローカルに留まり、別の受け入れられたターン内の pre-step の前にプロセスが終了すると失われる（[README の制限](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/README.md#known-limitations-and-deferred-work)）。

## 設定

```ts type-equiv
/** Deployment-owned plan guidance. */
interface PlanModeConfig {
  /** Guidance rendered as the `plan:policy` prompt section while plan mode is active. */
  section: string
}
```

`section` の欠如、空白、非文字列、および未知のキーは、無視されるのではなくプラグイン・ロード時に失敗する。プラン・モードがアクティブな間、`section` のテキストそのものが順序 50 で `plan:policy` の[システム・プロンプト・セクション](system-prompt.md)としてレンダリングされる。非アクティブなプラン・モードはテキストを提供しない。

## exit ツールと `/plan` コマンド

[`exit_plan_mode`](../../../docs/tool-catalog.md#deepseek-aidsh-plan-mode)は漸進的に開示される。選択されたプラン状態がアクティブである最初のステップからアドミットされ、Session の残りの間アドミットされたままなので、最初のプラン・モードへの突入がそれを要求のツール宣言に追加し、後の遷移はプランのガイダンスだけを変える。ネイティブ実行と PTC 実行は同じアドミット済みセットを使う。最初の突入前の呼び出しは `UNKNOWN_TOOL` を返し、プラン・モードがコミットされていない間の呼び出しは失敗する。プラン・モードでは、`#` 見出しで始まる完全な markdown プランを要求し、[user-questions seam](user-questions.md)を通じてレビューのために提示する。承認は `{ approved: true }` を返し、次に受け入れられたターン内の pre-step で追加される、静かな（ナレーションされない）保留中の exit を記録する。したがってプランのガイダンスは、assistant の現在のツール・バッチの残りの間アクティブなままであり、ツール結果自体が遷移を報告する。プランニングの継続はユーザーのフィードバックを運ぶ失敗した呼び出しなので、モデルは修正して再度提示する。却下されたレビューは `{ approved: false }` を返してターンを終えるので、プラン・モードはユーザーの次のメッセージまでアクティブなままである。相互作用チャネルの欠如とレビュー中のサービス・リロードも、暗黙にプラン・モードを離れるのではなく、呼び出しを失敗させる。

[`ctx.commands`](commands.md)が構成されているとき、プラグインは `/plan [off|message]` を登録する。素の `/plan` はプラン・モードを選択し、他の空でないメッセージはそれを選択した上で `agent.steer()` を通じてテキストを送信するので、プランのガイダンスの下で次のステップの通常の記録済みユーザー・メッセージになる。引数が正確に `off` の場合は非アクティブを選択し、保留中の突入を、それが追加されて要求に見えるようになる前にキャンセルする。

## サービス

`ctx.planMode` は記録されたプラン状態を所有し、ステップ開始時に選択された状態を適用してナレーションし、`plan:policy` セクション、`/plan` コマンド、条件付きの exit ツールを所有する。`get`／`set` のシグネチャは生成された[サービス・カタログ](#ctxplanmode--planmodecontroller)にある。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxplanmode--planmodecontroller"></a>

### `ctx.planMode` — `PlanModeController`

`ctx.planMode` は、記録されたプラン状態、ステップ開始時に選択された状態を適用してナレーションする処理、`plan:policy` セクション、`/plan` コマンド、条件付きの exit ツールを所有する。Client キャリアは投影の切り詰められた `{ active, pending }` ビューを公開する。投影リーダーが準備できていない間は agent のプロンプト・アセンブリが失敗するため、呼び出し元は agent を実行する前にプラグインのアクティブ化を待たなければならない。

```ts cordis-catalog
/**
 * Read the logged plan state and any selected state awaiting the next
 * accepted in-turn pre-step.
 *
 * @param agent The agent to read.
 * @returns Current logged state plus a pending selection, when present.
 */
get(agent: Agent): { active: boolean; pending?: boolean }

/**
 * Select whether plan mode should be active. Between turns the method
 * appends the change immediately because no in-turn pre-step will run until
 * another prompt starts a turn. The open-turn fold is the idle signal:
 * agent status stays `running` through post-turn checkpointing, when no
 * further in-turn pre-step runs. During an open turn the selection remains
 * pending until the next accepted in-turn pre-step. Repeated selection of
 * the current or already-pending state is a no-op.
 *
 * @param agent The agent to switch.
 * @param active Whether plan mode should be active.
 * @returns what happened: `committed` (logged now), `queued` (awaiting the
 * next accepted in-turn pre-step), `cancelled` (an opposite pending selection
 * was cleared; the logged state already matches), or `noop` (already in that
 * state).
 */
set(agent: Agent, active: boolean): 'committed' | 'queued' | 'cancelled' | 'noop'
```

型: [Agent](core.md)

ソース: [`packages/plan/plan-mode/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/src/index.ts)
<!-- END GENERATED cordis-surface -->
