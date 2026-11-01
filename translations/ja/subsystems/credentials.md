# ユーザー認証情報

日本語 | [English](../../../docs/subsystems/credentials.md) | [中文](../../../docs/subsystems/credentials.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/credentials.md](../../../docs/subsystems/credentials.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-credentials](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/credentials/credentials) の認証情報 seam は、シークレットを設定から切り離す。設定セクションと `cordis.yml` のエントリは*参照*（環境変数名）だけを運び、[dsh-credentials-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/credentials/credentials-local) のような provider が値を所有し、consumer は操作ごとに一度参照を解決する — LLM アダプタはモデルリクエストごとに一度解決するため、ローテーションされた認証情報は再起動なしで次のリクエストに届く。seam 全体の 1 つのルールがすべての provider を縛る。保存された値が空であれば、どこでも存在しないものとして扱う。

ソース: [`packages/credentials/credentials/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/credentials/src/index.ts)

## アイデンティティ

参照は 1 つの認証情報を POSIX 形式の環境変数名として名付ける。ブランドは、呼び出し元が認証情報参照を、パッケージやプロセス間で渡される他の文字列と混同するのを防ぐ。構築時にシェル識別子の構文を検証する。

```ts type-equiv
/** Nominal reference to one credential: a POSIX-style environment-variable name. */
type CredentialRef = Branded<'CredentialRef'>
```

<a id="resolution"></a>
## 解決

`resolve(ref)` は値と、それを供給した provider 定義のソース層を返す。未設定の間は `undefined` を返す。consumer は各操作で再解決し、操作をまたいでキャッシュしない — この操作ごとの読み取りがホットアップデートの仕組みである。

```ts type-equiv
/** One resolved credential value and the source layer that supplied it. */
interface ResolvedCredential {
  /** The non-empty secret value. */
  value: string
  /** Provider-defined source layer id (the local provider uses `env`, `file`, `project-env`, and `user-env`). */
  source: string
}
```

## 説明

`describe(ref)` は値を一切露出せずに設定サーフェスの問い合わせに答える。参照が解決するか、どの層からか、そして `set` が現在成功するかどうか。ローカル provider は管理対象ファイルを起動環境より上位に位置付けるため、値を保存すると常に有効な書き込み可能層が作成され、削除すると起動環境または `.env` のフォールバックが再び現れる。

```ts type-equiv
/**
 * Source and writability facts for one reference, safe for configuration UIs —
 * never the value. The view has no slot a value could ride in, which is what
 * lets the whole read half cross the Remote wire.
 */
interface CredentialInfo {
  /** Whether resolving the reference would currently return a value. */
  configured: boolean
  /** Source layer currently supplying the value; absent while unconfigured. */
  source?: string
  /** Whether the active provider can write this reference. */
  writable: boolean
}
```

## 変更のコミット

`credentials/reference-updated (ref)` は、provider 管理ソースへのコミット済み変更 — `set`、`unset`、またはストレージで観測された外部編集 — の後に発火する。プロセス環境側の変更は観測できず、決して発火しない。consumer はイベントを必要としない（操作ごとに再解決する）。これは「設定済み」バッジを更新する設定サーフェスのために存在する。

## 組み込み Platform 認証情報

PlatformSession は getPlatformSession による Host 専用スナップショットである。origin は設定済みの Platform issuer を名指し、token は保存されたアカウント認証情報を含む。userId は直近に成功した getProfile の安定したアカウント ID を繰り返し、成功するまでは、またはそのプロファイルが ID を持たない場合は null である。スナップショットは自身でプロファイルリクエストを発行せずにその ID を再利用するため、未知の ID は呼び出し元を遅延させる代わりに userId を null のままにする。安定した ID が初めて利用可能になるか変化したプロファイル読み取りは、`watch(signal, true)` を使う Host 購読者にのみ通知し、これによりアイデンティティ consumer はスナップショットを再読み取りできる。consumer は永続ブラウザストレージを origin と userId でキー付けし、null の場合は一時ストレージにフォールバックする。サインアウト済みアカウントと、認証情報読み取り中に変更された認証情報はスナップショットを返さない。issuer の不一致は失敗する。ネイティブ consumer は認証情報が変わったときのドキュメント無効化を所有する。このスナップショットは account-controller RPC、AccountView、AccountDetails から除外される。

AccountDetails.balance は、リチャージウォレットを value に、プロモーションウォレットを bonusWallets に投影し、それぞれ独立した通貨と小数の残高文字列を持つ。失敗したクエリにはウォレット配列が含まれない。

ボーナス通知クエリは、現在の Platform アカウント ID と対象注文をサーバー順で含む AccountBonusBatch を返す。AccountBonusNotification はサーバーメッセージと有効期限を保持し、認証情報は投影しない。確認応答は期待されるアカウント ID と注文 ID を運び、アカウント変更後は Host が拒否する。両方の通知操作は、言語クエリパラメータではなく x-client-locale を通じて開始元 UI の言語を使う。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxauthorization--authorizationservice"></a>

### `ctx.authorization` — `AuthorizationService`

`ctx.authorization`: 認証情報取得フローのレジストリ。キーごとに一度に 1 つの試行。

```ts cordis-catalog
/**
 * Offer a way to obtain one credential. One flow per key: two plugins
 * claiming the same key would each write a record in their own format, and
 * whichever ran last would leave the other reading a payload it cannot parse.
 *
 * @param flow - the key it writes, its label, its methods, and its runner.
 * @returns Disposer that withdraws this flow.
 * @throws {AuthorizationError} code `DUPLICATE_FLOW` when the key is already claimed.
 */
registerFlow(flow: AuthorizationFlow): () => void

/**
 * Every registered flow, for a surface listing what can be authorized.
 * @returns one entry per flow, in registration order.
 */
list(): readonly AuthorizationEntry[]

/**
 * One registered flow.
 * @param key - the credential record to ask about.
 * @returns the entry, or undefined when no flow claims that key.
 */
describe(key: CredentialKey): AuthorizationEntry | undefined

/**
 * Withdraw the attempt running for a key, if any. Separate from the
 * request's own signal because a request/response transport answers a Cancel
 * button on a second call, with no handle on the first one's signal.
 * @param key - the credential record whose attempt should stop.
 */
cancel(key: CredentialKey): void

/**
 * Run one attempt to authorize a key, and report how it ended.
 *
 * One attempt per key at a time. A second caller is refused rather than
 * joined: the two would be prompting different humans through the same flow,
 * and the second would answer questions the first was asked.
 *
 * @param request - the key, the method, the surface, and the cancel signal.
 * @returns `authorized` once the flow's record is committed during this
 *   attempt and observed, or `cancelled` when the human declined or the
 *   caller withdrew.
 * @throws {AuthorizationError} code `NO_FLOW` when nothing claims the key,
 *   `UNKNOWN_METHOD` when the named method is not one the flow offers,
 *   `ALREADY_IN_FLIGHT` when an attempt is already running for the key, or
 *   `NOT_COMMITTED` when the flow resolved without committing a record
 *   during the attempt.
 */
async begin(request: AuthorizationRequest): Promise<AuthorizationOutcome>
```

ソース: [`packages/credentials/authorization/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/authorization/src/index.ts)

<a id="ctxcredentials--credentialprovider-abstract-seam"></a>

### `ctx.credentials` — `CredentialProvider`（抽象 seam）

2 つのキー空間にわたる抽象認証情報サービスであり、2 つの問いに答える。

CredentialRef は「この環境変数名の背後に何があるか」に答える。まず provider 管理ストア、次にプロセス環境や `.env` ファイルのような環境のフォールバックである。この半分には seam 全体のルールが 1 つ縛る。保存された値が空であればどこでも存在しない — `resolve` はそれをスキップし、`describe` は未設定として報告する — ため、空欄が設定済みシークレットのふりをすることはない。

CredentialKey は「このプラグインがこの id に対してどの認証情報を保持しているか」に答える。ここでは何も層を重ねられない — 認可付与には読み取り元の環境がない — ため、レコードの存在がすべての事実であり、正しい書き込みは現在の値に依存するので modifyRecord が唯一の書き込み経路である（トークン更新は 1 つのロックの下での読み取り・判断・置換である）。

```ts cordis-catalog
/**
 * Resolve one reference to its current value. Resolution is per call:
 * consumers re-resolve at each operation and must not cache across
 * operations — that per-operation read is what makes a changed credential
 * reach the next operation without a restart.
 * @param ref - the reference to resolve.
 * @returns the value and its source, or `undefined` while unconfigured.
 */
abstract resolve(ref: CredentialRef): Promise<ResolvedCredential | undefined>

/**
 * Describe one reference for configuration surfaces without exposing the
 * value.
 * @param ref - the reference to describe.
 * @returns configured state, supplying source, and writability.
 */
abstract describe(ref: CredentialRef): Promise<CredentialInfo>

/**
 * Durably store one value in the provider-managed writable source. The stored
 * value takes precedence over ambient fallbacks, and empty values are
 * rejected (use {@link unset}).
 * @param ref - the reference to store.
 * @param value - the non-empty secret value.
 */
abstract set(ref: CredentialRef, value: string): Promise<void>

/**
 * Remove one reference from the provider-managed writable source; removing
 * an absent reference is a no-op. Ambient fallback layers may become
 * effective again after removal.
 * @param ref - the reference to remove.
 */
abstract unset(ref: CredentialRef): Promise<void>

/**
 * Read one stored record. The value is returned as its owner wrote it; a
 * {@link GrantRecord} payload is not interpreted on the way out.
 * @param key - the record to read.
 * @returns the record, or `undefined` while none is stored.
 */
abstract readRecord(key: CredentialKey): Promise<CredentialRecord | undefined>

/**
 * Describe one record for configuration surfaces without exposing its value.
 * @param key - the record to describe.
 * @returns presence, discriminant, and writability.
 */
abstract describeRecord(key: CredentialKey): Promise<CredentialRecordInfo>

/**
 * Enumerate every stored record's address and tag. Unlike the reference
 * half, which has no enumeration because configuration surfaces learn which
 * references exist from settings schemas, records have no such discovery
 * path: a surface that cannot list them cannot show what a user is
 * authorized for, nor find an orphan left by an uninstalled plugin.
 * @returns every stored record, values excluded.
 */
abstract listRecords(): Promise<readonly CredentialRecordEntry[]>

/**
 * Serialized read-modify-write over one record — the only write path.
 * `mutate` sees the record as it stands at the moment the write is
 * exclusive, and returning `undefined` leaves the entry untouched. Exclusion
 * holds across processes where the backing store supports it, which is what
 * makes a token refresh safe: two processes rotating one refresh token
 * concurrently would otherwise lose whichever wrote first.
 * @param key - the record to modify.
 * @param mutate - receives the current record and returns its replacement, or `undefined` to leave it.
 * @returns the record after the write, or the current one when `mutate` declined.
 */
abstract modifyRecord( key: CredentialKey, mutate: (current: CredentialRecord | undefined) => Promise<CredentialRecord | undefined>, ): Promise<CredentialRecord | undefined>

/**
 * Remove one record; removing an absent record is a no-op.
 * @param key - the record to remove.
 */
abstract deleteRecord(key: CredentialKey): Promise<void>
```

ソース: [`packages/credentials/credentials/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/credentials/src/index.ts)

<a id="ctxcredentialscontroller--credentialscontroller"></a>

### `ctx.credentialsController` — `CredentialsController`

生成された `ctx.remote.credentials` 名前空間を支える Host サービス。認証情報 seam 自体が持たないすべての wire 上の義務を担う。バッチのファンアウト上限、フィールドごとのビュー投影、参照文法のガード、拒否のマッピングである。シークレット値は一方向にしか渡らない — ここのどのメソッドも値を返さない。

```ts cordis-catalog
/**
 * Describe several references for one configuration surface. Batched because
 * a settings page describes every reference its rows name at once, and one
 * round trip keeps those rows from settling separately.
 * @param refs - reference names, at most {@link MAX_DESCRIBE_REFS}; a name outside the grammar
 *   rejects the whole call as `gateway/bad-request`.
 * @returns one view per requested name, keyed by that name.
 * @throws RemoteError when the request is invalid or no credential provider is mounted.
 */
@Remote async describe(refs: string[]): Promise<Record<string, CredentialInfo>>

/**
 * Store one value from a configuration surface. The value crosses the wire in
 * this direction only: no read path returns it.
 * @param ref - reference name to store under.
 * @param value - the non-empty secret value.
 * @throws RemoteError when the request is invalid, no provider is mounted, or the provider refuses the write.
 */
@Remote async set(ref: string, value: string): Promise<void>

/**
 * Remove one reference from a configuration surface.
 * @param ref - reference name to remove.
 * @throws RemoteError when the request is invalid, no provider is mounted, or the provider refuses the write.
 */
@Remote async unset(ref: string): Promise<void>
```

ソース: [`packages/api/settings-controller/src/credentials.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/settings-controller/src/credentials.ts)

<a id="ctxdeepseekaccount--deepseekaccount-abstract-seam"></a>

### `ctx.deepseekAccount` — `DeepSeekAccount`（抽象 seam）

アカウント操作。Host の consumer だけがリクエスト認証情報を取得できる。

```ts cordis-catalog
/**
 * Read stored-account presence and the latest login attempt.
 * @returns a snapshot without credentials or PKCE secrets.
 */
abstract getState(): Promise<AccountView>

/**
 * Query Platform profile independently of wallet balances.
 * A ready result whose stable profile ID first becomes available or changes notifies identity-enabled watch
 * consumers, so they re-read getPlatformSession; repeated IDs stay silent.
 * @param client - identity of the requesting UI for this call.
 * @returns profile outcome, or null if signed out or the grant changed during the query.
 */
abstract getProfile(client: AccountClientMetadata): Promise<AccountDetails['profile'] | null>

/**
 * Query Platform recharge and bonus wallet balances independently of profile data.
 * @param client - identity of the requesting UI for this call.
 * @returns balance outcome, or null if signed out or the grant changed during the query.
 */
abstract getBalance(client: AccountClientMetadata): Promise<AccountDetails['balance'] | null>

/**
 * Query the granted bonuses Platform has not yet recorded as displayed.
 * @param client - identity of the requesting UI for this call; its language selects the server-authored message.
 * @returns bonuses with their account, or null if signed out or the grant changed during the query.
 */
abstract getUnnotifiedBonuses(client: AccountClientMetadata): Promise<AccountBonusBatch | null>

/**
 * Record one displayed bonus as notified for the account it belongs to.
 * @param accountId - account the notification was read for; a different current account is never acknowledged.
 * @param orderId - granted bonus order the user saw.
 * @param client - identity of the requesting UI for this call.
 * @returns true once Platform records the acknowledgement; false if signed out or the account changed.
 */
abstract ackBonusNotified(accountId: AccountUserId, orderId: AccountBonusOrderId, client: AccountClientMetadata): Promise<boolean>

/**
 * Join an active attempt or start browser authorization.
 * @param client - identity of the requesting UI; a new attempt captures it, and joining retains the original attempt's identity.
 * @param callbackOrigin - browser-accessible loopback HTTP origin, including any SSH local port.
 * @param loginSource - initiating UI, used to return from a failed exchange.
 * @returns the initial snapshot without waiting for browser approval.
 */
abstract startSignIn(client: AccountClientMetadata, callbackOrigin: string, loginSource: 'web' | 'desktop'): Promise<AccountView>

/**
 * Cancel only the named attempt; committing attempts settle before returning.
 * @param id - attempt identity from this Host.
 * @returns state after cancellation or an already-started commit.
 */
abstract cancelSignIn(id: SignInAttemptId): Promise<AccountView>

/**
 * Remove the local grant while retaining API keys; the provider revokes it in the background.
 * @param client - identity of the requesting UI, captured for the background revocation retries.
 * @returns the signed-out state after local removal; remote failures never restore the grant.
 */
abstract signOut(client: AccountClientMetadata): Promise<AccountView>

/**
 * Subscribe to snapshots including a complete initial state.
 * @param signal - subscription lifetime; ending it never cancels login.
 * @param includeIdentityChanges - also notify when cached Platform identity changes; defaults to false.
 * @returns complete snapshots as account state changes.
 */
abstract watch(signal: AbortSignal, includeIdentityChanges?: boolean): AsyncIterable<AccountView>

/**
 * Resolve a credential only for the inference origin allowed by the provider.
 * @param url - actual request destination or API base URL.
 * @returns stored token, or undefined for other origins or a signed-out account.
 */
abstract resolveToken(url: string): Promise<string | undefined>

/**
 * Remove an inference-rejected token only while it still matches the stored login.
 * @param token - token captured by the rejected inference request.
 * @returns after matching credentials are removed and the expiry notification is emitted.
 */
abstract rejectToken(token: string): Promise<void>

/**
 * Read credentials for the configured Platform origin, bound to their issuing environment, and
 * pair them with the account ID from the last successful profile read; no profile request is made.
 * @returns a Host-only snapshot, or null while signed out or when the credential changed during the read.
 */
abstract getPlatformSession(): Promise<PlatformSession | null>

/**
 * Read existing login identity without creating a device or returning credentials.
 * @returns optional device/account identifiers and the provider's OS version string.
 */
abstract getDeviceIdentity(): Promise<{ deviceId?: string; userId?: AccountUserId; osVersion: string }>
```

ソース: [`packages/credentials/deepseek-account/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/deepseek-account/src/index.ts)

<a id="authorization-events"></a>

### `authorization/*` イベント

<a id="authorizationsettled--emit"></a>

#### `authorization/settled` — emit

1 つの認可試行が終了し、そのキーを解放した。失敗を含むすべての終端結果で発火するため、開始していないキーを監視しているサーフェス（2 つ目のブラウザタブなど）は試行が終わったことを知る。

```ts cordis-catalog
/**
 * One authorization attempt has finished and released its key. Fires for
 * every terminal outcome, failures included, so a surface watching a key it
 * did not start (a second browser tab) learns the attempt is over.
 * @mode emit
 * @param key - the credential record the finished attempt was authorizing.
 * @param settlement - how it ended, including the `failed` case its caller sees as a thrown error.
 */
'authorization/settled'(key: CredentialKey, settlement: AuthorizationSettlement): void
```

ソース: [`packages/credentials/authorization/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/authorization/src/index.ts)

<a id="credentials-events"></a>

### `credentials/*` イベント

<a id="credentialsrecord-updated--emit"></a>

#### `credentials/record-updated` — emit

保存された認証情報レコードへのコミット済み変更。書き込みを行った `modifyRecord`、削除を行った `deleteRecord`、またはストレージで観測された外部編集である。`credentials/reference-updated` とは別である。2 つのキー文法は互いに素であり、1 つのイベントで両方を受け取ったリスナーは、対象がどちらの空間に属するか判断できない。リスナーの失敗は `credentials/reference-updated` と同じ条件で封じ込められる。

```ts cordis-catalog
/**
 * Committed change to a stored credential record: a `modifyRecord` that
 * wrote, a `deleteRecord` that removed, or an external edit observed in
 * storage. Separate from `credentials/reference-updated` because the two key
 * grammars are disjoint — a listener that received both on one event could
 * not tell which space a subject belongs to. Listener failures are
 * contained on the same terms as `credentials/reference-updated`.
 * @param key - the record whose stored value changed.
 * @mode emit
 */
'credentials/record-updated'(key: CredentialKey): void
```

ソース: [`packages/credentials/credentials/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/credentials/src/types.ts)

<a id="credentialsreference-updated--emit"></a>

#### `credentials/reference-updated` — emit

provider 管理の認証情報ソースへのコミット済み変更。`set`、`unset`、またはストレージで観測された外部編集である。プロセス環境側の変更は観測できず、決して発火しない。リスナーの失敗（同期 throw と非同期 reject の両方）は、コミットされた操作の結果を変えずに封じ込められ、ログに記録される。

```ts cordis-catalog
/**
 * Committed change to a provider-managed credential source: a `set`, an
 * `unset`, or an external edit observed in storage. Ambient
 * process-environment changes are not observable and never emit. Listener
 * failures are contained and logged — a sync throw and an async rejection
 * alike — without changing the committed operation's outcome.
 * @param ref - the reference whose stored value changed.
 * @mode emit
 */
'credentials/reference-updated'(ref: CredentialRef): void
```

ソース: [`packages/credentials/credentials/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/credentials/src/types.ts)

<a id="deepseek-account-events"></a>

### `deepseek-account/*` イベント

<a id="deepseek-accountmodel-sign-in-required--emit"></a>

#### `deepseek-account/model-sign-in-required` — emit

アカウントモデルのリクエストがユーザーのサインインを必要とする。

```ts cordis-catalog
/** An account model request requires the user to sign in.
 * @mode emit
 */
'deepseek-account/model-sign-in-required'(): void
```

ソース: [`packages/credentials/deepseek-account/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/deepseek-account/src/types.ts)

<a id="deepseek-accountsession-expired--emit"></a>

#### `deepseek-account/session-expired` — emit

サーバーの拒否により現在のアカウント認証情報が削除された。この通知は再送されない。

```ts cordis-catalog
/** Server rejection removed the current account credential; this notification is not replayed.
 * @mode emit
 */
'deepseek-account/session-expired'(): void
```

ソース: [`packages/credentials/deepseek-account/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/deepseek-account/src/types.ts)

<a id="deepseek-accountsigned-out--emit"></a>

#### `deepseek-account/signed-out` — emit

ローカル付与の削除が完了した。

```ts cordis-catalog
/** Local grant removal has completed.
 * @mode emit
 */
'deepseek-account/signed-out'(): void
```

ソース: [`packages/credentials/deepseek-account/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/deepseek-account/src/index.ts)
<!-- END GENERATED cordis-surface -->

アカウント Service Definition は getState、getProfile、getBalance、getUnnotifiedBonuses、ackBonusNotified、startSignIn、cancelSignIn、signOut、watch、および Host 専用の resolveToken と getPlatformSession を公開する。platform provider は AuthorizationFlow とプライベートな GrantRecord でそれを実装する。AccountView は保存された存在とサーバー検証を区別し、試行 ID はキャンセルを 1 つのローカルフローに結び付ける。[account パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/credentials/deepseek-account/README.md)を参照。

`AccountClientMetadata` は呼び出し元の `DSH_CLIENT_VERSION` からの `version`、アクティブな UI の `locale`、操作開始時にサンプリングした `timezoneOffsetSeconds` を運ぶ。オフセットはローカル時刻から UTC を引いた秒数で、UTC+8 は `28800` である。認証情報は含まない。ログイン試行は交換とキャンセルを通じて開始時のメタデータを保持し、サインアウトは失効リトライのためにそのメタデータを保持する。
