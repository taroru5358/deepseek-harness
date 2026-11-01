# クライアント・リソース

日本語 | [English](../../../docs/subsystems/client-resources.md) | [中文](../../../docs/subsystems/client-resources.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/client-resources.md](../../../docs/subsystems/client-resources.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

クライアント・リソース・モデルは、アドレスをあらゆる Web Client コンポーネントのためのライブ・データに変える。[`dsh-client-resources`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md)は `ctx.resources` サービスと `useResource` グローバル標準フックを提供する。ある種のコンテンツを所有するパッケージは、その**プロトコル**に対して 1 つの**プロバイダー**を登録し、コンポーネントは所有者のランタイムをインポートせずに**アドレス**でコンテンツの現在の状態を読む。右 Sidebar のタブはこのモデルの最初のコンシューマーである（[右 Sidebar](sidebar-right.md)）。決定記録は[クライアント・リソース・モデルの Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-05-client-resource-model.md)である。

このページは開発者向けリファレンスである。アドレスの書き方、プロバイダーの登録方法、リソースの読み方、状態と失敗の意味、モデルがリソースを保持し解放する方法を扱う。

## アドレス

リソース・アドレスは `dsh-resource://<type>/…` の URL である。ホストがプロトコルを指名し、`ResourceProtocolMap` のキーでなければならない。パスはプロトコル自身のもので、その所有者が各セグメントをパーセント・エンコードする。スコープを必要とするプロトコルはそれをパスに置く。`file` プロトコルのアドレスは `dsh-resource://file/session/<sessionId>/<path>` と読み、ここで path はワークスペース相対または絶対で、先頭のスラッシュは保持される。これは `fileAddressFor(sessionId, cwd, path)` で組み立て、[`dsh-util-workspace-path`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/workspace-path/README.md)の `parseFileAddress(address)` で読み戻す。モデル自身はスキームとホストだけを読む。`protocolOf(address)` は `dsh-resource://` URL の小文字化されたホストを返し、それ以外には `undefined` を返す。他のスキームのアドレス（Sidebar の `sidebar://guide`）はリソースを指名せず、`none` として読まれる。

| アドレス | プロトコル・キー | 読み取り結果 |
|---|---|---|
| `dsh-resource://file/session/s1/notes/a.md` | `file` | `file` プロバイダーが登録されている場合、セッション `s1` のワークスペース・ルート下の `notes/a.md` のメタデータ |
| `dsh-resource://file/absolute/home/me/notes.md` | `file` | パース可能だが `workspace-file/unknown-workspace` で失敗する。認可する Session がなく、現在の Session も Tab の Session も借用されない |
| `DSH-RESOURCE://File/session/s1/a` | `file` | 別個のレコード。アドレスは文字列として比較され、`openResource` は `fileAddressFor` が出力する正規の小文字綴りのみを受け入れる |
| `dsh-resource://subagentchat/session/c1?parent=p1&mode=continuable` | `subagentchat` | アドレス指定されたサブ agent の Conversation。その値はリソースが閉じるまで `SessionReference` を所有する |
| `sidebar://guide` | — | `none`。ナビゲーション・アドレス |
| `/home/me/notes.md` | — | `none`。URL ではない |

## プロバイダーの登録

プロトコルの所有者は `ResourceProtocolMap` にその値の型を宣言し、自身の `ctx.effect` 内で 1 つのプロバイダーを登録するので、プロトコルはプラグインと同じ寿命だけ生きる（[プロトコルの提供](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#provide-a-protocol)）。`open(address, { signal })` は `RemoteResult` フレームのストリームを返す。最初が現在の状態で、その後は変化ごとに 1 フレームである。`signal` が中断したら停止しなければならない。失敗は `ok: false` のフレームで、`RemoteFailure` を運ぶ。ストリーム内での throw はプログラミング・エラーであり、捕捉されない。

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'
import type { RemoteResult } from '@deepseek-ai/dsh-typert-protocol'
import type {} from '@deepseek-ai/dsh-client-resources/client'

interface NoteView { readonly title: string; readonly updatedAt: string }

declare module '@deepseek-ai/dsh-client-ui-slots' {
  interface ResourceProtocolMap { note: NoteView }
}

export const inject = ['resources', 'remote']

export function apply(ctx: Context): void {
  ctx.effect(() => ctx.resources.register<'note'>({
    protocol: 'note',
    async *open(address, { signal }): AsyncIterable<RemoteResult<NoteView>> {
      const id = new URL(address).pathname.slice(1)
      yield await ctx.remote.notes.read(id, signal)
      for await (const change of ctx.remote.notes.follow(id, signal)) yield change
    },
  }), 'my-notes: note resource provider')
}
```

プロトコルにはプロバイダーがちょうど 1 つある。2 つ目の登録は throw する。そのプロトコルのアドレスがすでに保持されている間に登録すると、それらのストリームがすぐに開かれる。プロバイダーを破棄するとそれらのストリームが終了し、プロバイダーが戻るまでアドレスは `none` として読まれる。

## リソースの読み取り

すべての Slot コンポーネントは、スコープに関係なく props で `useResource` を受け取る（[Slots](slots.md)）。`useResource<P>(address)` はプロトコルを型引数として指名し、そのアドレスの現在のスナップショットを返す。購読することがリソースを開いたまま保持するのであり、別の保持者がリソースを生かしている間にマウントするコンポーネントは、ストリームを再度開かずに最新の値をすぐに読む（[リソースの読み取り](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#read-a-resource)）。

| `status` | 意味 | `value` | `failure` |
|---|---|---|---|
| `none` | アドレスのプロトコルにプロバイダーが登録されていない、またはアドレスがリソース・アドレスではない | `undefined` | `undefined` |
| `loading` | プロバイダーのストリームが開いており、まだ yield していない | `undefined` | `undefined` |
| `live` | 最新のフレームが成功した | 最新の `ok` 値 | `undefined` |
| `failed` | 最新のフレームが失敗を報告した | 最後の `ok` 値（保持される） | そのフレームの `RemoteFailure` |

```tsx ignore-check
import type { PropsRuntime } from '@deepseek-ai/dsh-client-ui-slots'
import type {} from '@deepseek-ai/dsh-api-workspace-files/client'

type Props = PropsRuntime<'sidebar.right.pane.tab'>

export function FileHeader({ useTabInfo, useResource, t }: Props) {
  const { tab } = useTabInfo()
  const meta = useResource<'file'>(tab.contentId)
  if (meta.status === 'failed') return <p role="alert">{t('failed', { code: meta.failure.code })}</p>
  return (
    <header>
      {tab.title}
    </header>
  )
}
```

`failed` はコンシューマー自身が提示する。モデルは失敗のそばに最後の値を保つので、ボディは空白ではなく通知付きで陳腐化したコンテンツを表示でき、次の `ok` フレームが失敗をクリアする。モデル内の何もユーザー可視のテキストを生成しない。

## 保持と解放

リソースは保持者を持つ間、すなわち購読された `useResource` かピンの間、生きている。`ctx.resources.pin(address, signal)` は `signal` が中断するまで購読せずにリソースを開いたままにし、すでに中断されたシグナルは何もピンしない。右 Sidebar は開いている各タブ・レコードのアドレスをそのレコードの寿命の間ピンするので、タブを切り替えるとボディはアンマウントされるがストリームは閉じない。最初の保持者がプロバイダーのストリームを開き、最後の解放がそれを中断し、値を破棄し、スナップショットを `loading`（プロバイダーあり）または `none`（なし）に戻す。その解放後にプロバイダーが yield したフレームは破棄され、イテレーターは返される。`ctx.resources.source(address)` はフックの背後にある素のオブザーバブルで、アドレスごとに参照が安定しており、React の外の呼び出し元向けである。そのスナップショットを読んでもリソースは保持されない（[ライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#lifecycle)）。

ほとんどのストリームはコンテンツではなくメタデータを運ぶ。`file` プロバイダーの値は `WorkspaceFileStat { absolutePath, version, bytes? }` である。最初のフレームは Host の `stat` から来て、その後の観測がバージョンを更新する。コンシューマーは Workspace Files の Remote 名前空間を通じてコンテンツを読み、Preview はタブごとに独立してリフレッシュを所有する（[`dsh-api-workspace-files`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md)）。`subagentchat` プロバイダーはアドレス指定された Session を保持し、その `SessionReference` を yield し、リソースのシグナルが中断したときにその参照を解放する。

## 制限

レコードはページの寿命だけ生きる。アドレスのレコードは最後の保持者が去った後も残り、ストリームも値も保持しないため、メモリはこれまでに読まれた個別アドレスの数だけ増える。`signal` を無視するプロバイダーは次のフレームまで実行し続ける。失敗の型は Remote フェイスの `RemoteFailure` であるため、ソースが Remote 呼び出しでないプロバイダーはそれを自分で生成する。綴りを間違えたプロトコルや不正なアドレスは、他の診断なしで `none` として読まれる。
