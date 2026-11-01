# クックブック: Remote API の追加

日本語 | [English](../../../docs/cookbook/adding-a-remote-api.md) | [中文](../../../docs/cookbook/adding-a-remote-api.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-remote-api.md](../../../docs/cookbook/adding-a-remote-api.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`ctx.remote` エンドポイントの追加や変更は、このページの5つのステップで行います。メソッドを宣言し、その失敗を宣言し、パッケージに登録し、Client 側で消費し、テストします。デコレーターのセマンティクス、lookup の解決、生成パイプライン、`/api` ルートはメカニズムであり、[API Gateway リファレンス](../api-gateway.md) に属します。このページは各ステップの作業と、それが満たすべき規約を示します。プログラミング・インターフェースがこの形になっている理由は [Typert Remote メソッド呼び出し Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.md) に、失敗が1つの `RemoteError` とコード表になる理由は [失敗語彙 Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-28-ctx-remote-failure-vocabulary.md) にあります。

## 1. API を宣言する

オーナーは Host 側の Cordis サービスです。`TypertRemoteService` を拡張してサービス・キーとワイヤー名前空間を結び付け、公開するメソッドを `@Remote` でマークします。シグネチャがすでにワイヤーの規約を満たしているなら、ビジネス・メソッド自体をマークします。形を変える必要がある場合（`signal` の追加、パラメーターの並べ替え、別名でのエクスポート）にだけ `remoteExport*` アダプターを書き、そのアダプターから改名していないビジネス・メソッドを呼びます。lookup オブジェクト（`Agent`、`Session`）はトップレベルのパラメーター位置にのみ置けます。協調的キャンセルをサポートするメソッドは、最後のパラメーターとして `signal: AbortSignal` を取ります。

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { Agent } from '@deepseek-ai/dsh-agent'
import { Remote, TypertRemoteService } from '@deepseek-ai/dsh-typert-protocol'

/** One stored note as a Client reads it. */
export interface NoteRow {
  readonly noteId: string
  readonly title: string
}

declare module '@deepseek-ai/cordis' {
  interface Context {
    notesController: NotesController
  }
}

export class NotesController extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'notesController', { namespace: 'notes' })
  }

  /**
   * @param agent - lookup parameter the Gateway resolves from its wire identity.
   * @param signal - carrier cancellation, always the final parameter.
   * @returns the notes this Agent's session owns.
   */
  @Remote('list')
  async remoteExportList(agent: Agent, signal: AbortSignal): Promise<NoteRow[]> {
    return await this.list(agent, signal)
  }

  /** The in-process API the adapter above delegates to, unchanged by it. */
  async list(agent: Agent, signal: AbortSignal): Promise<NoteRow[]> {
    signal.throwIfAborted()
    return await Promise.resolve([{ noteId: `${agent.id}-1`, title: 'draft' }])
  }
}
```

## 2. 失敗を宣言する

Remote の失敗は `RemoteError` という1つのクラスです。宣言マージでドメインのコードを `RemoteErrorDetailsMap` にマージし、失敗箇所で `throw new RemoteError(code, message, details)` を実行します。ドメイン・エラー・クラスの一族を作らないでください。exit マッピング関数も書きません。このエンドポイントと無関係な例外は事前分類されません。Gateway がそれを `gateway/internal` に畳み込むからです。任意のプロバイダー例外を1つのドメイン・コードに分類する場合にだけ `catch` を書き、元の例外を `cause` として付けます。

コードは `<domain>/<reason>` の形をとり、その宣言には4つの配置ルールがあります。

- プロデューサーが1つだけ: 生成するパッケージ内の throw の隣で宣言します。
- 複数のパッケージが生成する: 両方が依存する最も下位のドメイン・パッケージで宣言します（`core/session` の `session/not-found`、`dsh-workspace` の `workspace/not-found`）。
- キャリア・コード `gateway/bad-request`、`gateway/cancelled`、`gateway/internal` は protocol で宣言され、Gateway のインフラストラクチャ・コードは gateway で宣言されます。それらを使い、決してコピーしないでください。
- ワイヤーを越えないローカルな失敗はコード表に入れません。呼び出し元自身の型で表現します。

```ts
import { RemoteError } from '@deepseek-ai/dsh-typert-protocol'

declare module '@deepseek-ai/dsh-typert-protocol' {
  interface RemoteErrorDetailsMap {
    /** No stored note carries that id. */
    'note/not-found': { readonly noteId: string }
    /** The store refused an otherwise valid write. */
    'note/rejected': { readonly noteId: string }
  }
}

declare const stored: ReadonlyMap<string, string>
declare function persist(noteId: string, title: string): Promise<void>

export async function rename(noteId: string, title: string): Promise<void> {
  if (!stored.has(noteId)) {
    throw new RemoteError('note/not-found', `no note "${noteId}"`, { noteId })
  }
  try {
    await persist(noteId, title)
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : String(error)
    throw new RemoteError('note/rejected', message, { noteId }, { cause: error })
  }
}
```

## 3. パッケージに登録する

`@Remote` は Loader エントリーのプラグイン・パッケージに置く必要があります。オーナーが抽象シームの場合、コントローラーは `packages/api/` 以下の対応するパッケージに入ります。マニフェストには生成された2つのエントリーと protocol の peer 依存が加わり、Client 側では `@deepseek-ai/dsh-api-remotes` のアセンブリがコントリビューションをマウントし、コンシューマーが必要とする型語彙を再エクスポートします。各エントリーが指す生成物と、生成パイプラインの順序は [API Gateway リファレンス](../api-gateway.md) にあります。

```json
{
  "exports": {
    "./typert": { "types": "./lib/typert.host.d.ts", "default": "./lib/typert.host.js" },
    "./remote": { "types": "./lib/typert.remote-client.d.ts", "default": "./lib/typert.remote-client.js" }
  },
  "peerDependencies": { "@deepseek-ai/dsh-typert-protocol": "workspace:*" },
  "devDependencies": { "@deepseek-ai/dsh-typert-protocol": "workspace:*" }
}
```

シグネチャ、コード表、名前空間、エクスポート名を変更したら `pnpm run build:lib` を再実行します。Client に新しい宣言とコーデックを渡すのはこのコマンドだからです。実装本体だけの変更には再生成は不要です。

## 4. Client で消費する

呼び出す側のプラグインは `inject` に `remote` と `remote.<namespace>` の両方を宣言し、呼び出し箇所では `ctx.remote.<namespace>.<method>(...)` を直接書きます。`Pick<ClientRemote, …>` による絞り込みも、手書きのメソッド・シグネチャも、ワイヤー中継オブジェクトも不要です。結果は `RemoteResult<T>` なので、その場で `if (!result.ok)` により分岐し、`instanceof` ではなく `code` で判別します。code の分岐はそれ自体で `details` を絞り込みます。例外フローの箇所では `throw result.error` を書きます（本物の Error です）。受け取った側は `isRemoteFailure` を使って Remote の失敗とローカルな欠陥を区別し、欠陥は再 throw します。防御的な catch は書かないでください。Remote 呼び出しは reject せず、アセンブリの誤りはクラッシュすべきです。

固定の Host 情報は `ctx.remote.$host` から取得します。`home` と `isLoopback` は購読も世代カウンターもない単純な読み取りで、`home` は最初の ready フレームまで `undefined` です。再接続後は `ctx.on('connection/reset')` またはドメイン自身の remote イベントを通じて更新します。呼び出し元が unary 呼び出しを中断すると、結果は throw ではなくエラー分岐上の `gateway/cancelled` になります。

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'
import { isRemoteFailure } from '@deepseek-ai/dsh-api-gateway/client'
import type {} from '@deepseek-ai/dsh-api-remotes/client'

export const inject = ['remote', 'remote.notes']

declare const ctx: Context

/** Store-side read: the error branch is handled where the code is meaningful. */
export async function noteTitles(): Promise<readonly string[]> {
  const result = await ctx.remote.notes.list()
  if (!result.ok) {
    if (result.error.code === 'note/not-found') return []
    throw result.error
  }
  return result.value.map(row => row.title)
}

/** Action-side: a Remote failure becomes copy; a local fault keeps crashing. */
export async function renderTitles(): Promise<string> {
  try {
    return (await noteTitles()).join(', ')
  } catch (error: unknown) {
    if (!isRemoteFailure(error)) throw error
    return `unavailable (${error.code})`
  }
}

/** Fixed Host facts as plain reads. */
export function hostLabel(): string {
  const { home, isLoopback } = ctx.remote.$host
  return home ?? (isLoopback ? 'local host' : 'remote host')
}
```

## 5. テストする

オーナー側では、throw されたコードをアサートします。catch した後に `remoteErrorOf` で失敗を取り出し、`code` と注目する details フィールドを `toMatchObject` で比較します。エラー・オブジェクトを `toEqual` で深く比較したり、`instanceof` をアサートしたりしないでください。

```ts
import { remoteErrorOf } from '@deepseek-ai/dsh-typert-protocol'
import { expect, it } from 'vitest'

declare function rename(noteId: string, title: string): Promise<void>

it('refuses an unknown note before writing', async () => {
  const failure = await rename('n-404', 'fresh title').catch((error: unknown) => error)

  expect(remoteErrorOf(failure)).toMatchObject({
    code: 'note/not-found',
    details: { noteId: 'n-404' },
  })
})
```

Client 側のダブルは実インスタンスを返します。`RemoteError` と `TestRemote` の値インポートは `@deepseek-ai/dsh-client-test-runtime` から取得してください。`api-remotes` ファサードからの値インポートは未ビルドのアセンブリ連鎖をロードしてしまうからです。`TestRemote.$host` は spec が直接代入できる単純なフィールドです。

```ts ignore-check
import { Context } from '@deepseek-ai/cordis'
import { RemoteError, TestRemote } from '@deepseek-ai/dsh-client-test-runtime'
import { expect, it } from 'vitest'

it('renders the failure code the Host reported', async () => {
  const ctx = new Context()
  const remote = new TestRemote(ctx, {
    notes: {
      list: () => Promise.resolve({
        ok: false as const,
        error: new RemoteError('note/not-found', 'no note "n-404"', { noteId: 'n-404' }),
      }),
    },
  })
  remote.$host = { home: '/home/fixture', isLoopback: true }

  await expect(ctx.remote.notes.list()).resolves.toMatchObject({ error: { code: 'note/not-found' } })
})
```

## 検証

1. `pnpm run build:lib`: シグネチャ、コード表、名前空間、エクスポート名のいずれかを変更したら必須です。Client の宣言とコーデックを生成するからです。
2. `pnpm run typecheck`: Host と Client の両方のプログラムが対象で、到達不能なパッケージにマージされたコードは赤になります。
3. 両側の spec を名前で実行します: `npx vitest run <owner spec> <client spec>`。
4. エンドポイントが製品から見える面に到達する場合は、[テストポリシー](../testing.md) に従って recorded-session スナップショットを追加します。
