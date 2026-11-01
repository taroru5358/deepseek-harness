# ストレージ

日本語 | [English](../../../docs/subsystems/storage.md) | [中文](../../../docs/subsystems/storage.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/storage.md](../../../docs/subsystems/storage.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ストレージサブシステムは、セッションイベントログではないすべてを永続化する（セッションログには独自の seam がある — [persistence.md](persistence.md)）。これは 1 つの任意 capability であり、agent-loop の背骨の一部ではなく、[capability seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-06-13-capability-seams.md) として分割される: ハブと Service Definition（[dsh-storage](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/storage/storage)、`ctx.storage`）、Service Providers（[dsh-storage-json](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/storage/storage-json)、`json` として登録、および [dsh-storage-sqlite](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/storage/storage-sqlite)、`sqlite` として登録）、そして Consumer データフォーム（[dsh-storage-domain](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/storage/storage-domain)、`ctx.storageDomain`、`ctx.storage.domain` としても到達可能） — バックエンド契約の唯一の Consumer であり、他のすべてが使う型付き API である。ハブ自体は IO を行わない: バックエンドが媒体を、データフォームが意味論を所有し、製品パッケージはバックエンドに直接触れない。設計記録: [domain KV storage Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.md)。

ソース: [`packages/storage/storage/src/backend.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage/src/backend.ts) · [`packages/storage/storage-domain/src/spec.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-domain/src/spec.ts) · [`packages/storage/storage-domain/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-domain/src/events.ts)

## ハブ: `ctx.storage`

`Storage`（[シグネチャ](#ctxstorage--storage)）はストアではなく合流点である。`ctx.storage.backend` は名前 → バックエンドのテーブルである: 複数のバックエンドが並んでマウントされたままで、どのバックエンドがどのコンシューマーにサービスするかはそのコンシューマーの設定（ドメイン層のルートテーブル）であり、ハブ全体の選択ではない。`register(name, backend)` は disposer を返す。重複した名前と未知のルックアップは `StorageError` を投げる。破棄は名前の登録解除だけを行う — 所有プラグインが登録解除後にバックエンドを閉じる。各バックエンドプラグインはライフサイクル専用のサービスキー（`storageBackendServiceKey(name)`）も公開し、フォーム provider がそれを注入することで、それらのアクティベーションがバックエンド登録と競合しない。

データフォームはマージ拡張可能なキーマップの下でハブにマウントされる:

```ts type-equiv
/**
 * Data forms mountable on the hub, keyed by form name. Form owners extend
 * this map via declaration merging (the domain layer merges
 * `domain: DomainFacility`) and mount the facility in their `apply`.
 */
interface StorageForms {}
```

`mount(form, facility)` は効果であり、その disposer がアンマウントする。同じキーの 2 回目のマウントは `duplicate-mount` を投げる。`form(form)` はマウントされたファシリティを解決し、所有プラグインがロードされるまで `form-not-mounted` を投げる — アセンブリは暗黙に遅延させるのではなく、それに応じてプラグインを順序付ける。ドメイン層は `domain: DomainFacility` をマージするため、`ctx.storage.domain` と `ctx.storageDomain` は同じオブジェクトである。

## バックエンド契約

```ts type-equiv
/**
 * One registered backend. A backend owns exactly one medium and shares its
 * lifecycle across all facets; facets are optional members — a backend that
 * cannot serve a data kind simply omits it, and resolution fails loud instead.
 */
interface StorageBackend {
  /** Key-value operations; absent when this backend cannot serve them. */
  readonly kv?: KvFacet

  /**
   * Drain in-flight writes across all open units and release the medium.
   * Idempotent; concurrent and repeated calls resolve once teardown finishes.
   * @returns resolution after the medium is released.
   */
  close(): Promise<void>
}
```

バックエンドは 1 つの媒体（ファイルツリーのルート、データベースファイル）を所有し、任意の操作グループを公開する。`kv` が出荷される唯一のグループである。`KvFacet.open(descriptor)` は 1 つの名前付きユニットを開く — `KvUnitDescriptor` は名前、現在のフォーマットバージョン、任意の互換バージョン、テーブル名、グローバルシングルトンスロットの有無を運ぶ — そして `loadAll`、`putRecord`、`deleteRecord`、`deleteRecords`、`setGlobal`、`close` を持つ `KvUnit` を返す。ユニット名とテーブル名は `UNIT_NAME_RE` に一致しなければならない（ファイル名としても SQL 識別子セグメントとしても安全）。レコードキーは `single` レイアウトでは不透明な文字列であり、`per-record` レイアウトでは `[a-zA-Z0-9_-]+` のパスセグメントに一致する。ユニットは並行書き込みを直列化しない — 順序付けは呼び出し元に属する — ドキュメント全体と行ストアの呼び出しは媒体上でアトミックだが、解決前に中断された per-record バッチは部分的に適用されることがある。解決された書き込みは永続的であり、per-record の削除は POSIX でディレクトリ変更を同期する。`deleteRecords` はバッチ全体を 1 回の呼び出しで適用する: リストされたすべてのレコードが読み取り可能な状態から存在せず、バックエンドがバッチを受け入れた時点で解決し、拒否は通常バッチを未適用のままにする。バッチの状態を確立または復元できないバックエンドは `unknown-state` で拒否し、その後そのユニットに対するすべての後続操作は `close()` が解放するまで同じエラーで拒否する（close 後の呼び出しは `closed` を拒否する）。アトミック置換後の `single` レイアウトの失敗（ディレクトリ fsync エラー）は、呼び出し元が拒否を見る一方で新しいファイルを媒体上に残すことがある。バックエンドとそのドメインを閉じ、再オープンして保存状態を検査してから再試行する。現在および宣言された互換バージョンの外でスタンプされた `single` 媒体は `version-mismatch` を拒否する。受け入れられた集合の外でスタンプされた `per-record` ドキュメントは存在しないものとして読まれる。ユニットとして解析できない媒体は `malformed-medium` を拒否する。[`backend.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage/src/backend.ts) は規範的な条項ごとの契約であり、[`tests/contract.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage/tests/contract.ts) の共有適合スイートが共有条項を各バックエンドに対して検査する。[json バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-json/README.md) はユニットごとに 1 つの人間可読なファイル全体をアトミックに再公開する。[sqlite バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-sqlite/README.md) は頻繁に更新されるデータのために 1 つのデータベース内の行ごとに 1 つのドキュメントを保存する。

## ドメインの宣言

ドメインは所有パッケージによって spec オブジェクトとして 1 回宣言される — ドメインの同一性、レイアウト、レコードスキーマの唯一の源である（zod なので `z.infer` がコンシューマー型の重複を防ぐ）:

```ts type-equiv
/** Static declaration of one domain: identity, version, and record layout. */
interface DomainSpec {
  /** Domain name; must match `UNIT_NAME_RE` (doubles as the backend unit name). */
  readonly name: string
  /** Current domain format version; reads enforce it according to the selected layout. */
  readonly version: number
  /**
   * Document layout for the backend unit: `single` (the default) stores the
   * whole unit as one document; `per-record` stores each record as its own
   * document, for units whose records are large, sparse, or individually
   * disposable — the projection cache — with per-record version checks.
   * Row stores retain unit-wide versioning for either layout request.
   */
  readonly layout?: 'single' | 'per-record'
  /**
   * Older domain versions whose stored records the current record schemas
   * also accept (the declaring owner vouches for that, typically by
   * declaring the fields older records lack as optional). Ordinary reads admit
   * listed versions without restamping saved data; writes stamp {@link version}
   * while leaving untouched records as saved. Current schemas must therefore
   * accept mixed generations even under the current unit stamp.
   */
  readonly compatibleVersions?: readonly number[]
  /**
   * What `open` does with a stored table record that fails its zod schema.
   * Absent (the default), the whole open rejects with `invalid-record` —
   * right for authoritative data. `'backup-and-skip'` is for domains whose
   * records are disposable derived data: the backend moves the record's
   * document aside (`KvUnit.backupRecord`), the failure is logged with
   * its cause, and the open continues with the record absent. A backend
   * without `backupRecord` (no per-record document to move) falls back
   * to the rejecting default. The global slot always rejects.
   */
  readonly invalidRecords?: 'backup-and-skip'
  /** Optional global singleton slot. */
  readonly global?: DomainGlobalSpec<unknown>
  /** Table declarations keyed by table name; each name must match `UNIT_NAME_RE`. */
  readonly tables: Record<string, DomainTableSpec>
}
```

`defineDomain(spec)` は spec のリテラル型を固定し、媒体に触れる前に所有モジュールのロード時に大声で失敗する: `UNIT_NAME_RE` の外のドメインまたはテーブル名、非負整数でないバージョン、または `null` を受け入れるグローバルスキーマはすべてスローする（`null` は媒体の「決して書かれていない」センチネルなので、保存された nullable グローバルはラウンドトリップできない）。`domainTable<K, V>(schema)` はファントムなコンパイル時キー型（通常は [ブランド付き id](core.md#branded-ids)）を持つ 1 つのテーブルを宣言する。`descriptorOf(spec)` はバックエンド向けユニット記述子を投影する。

<a id="domain-versioning"></a>
## ドメインバージョニング

`version` は現在の読み書きフォーマットを名付ける。`compatibleVersions` は、所有者が現在のスキーマでレコードとグローバル値をデコードできる古いバージョンを列挙する。`version` 未満の非負整数のみが許可される。その宣言がなければ、読み取りは正確なバージョンのままである。受け入れはレコード検証を弱めず、より新しい未知のバージョンを許可もしない。各行の保存フィールドはその意味を一意に決定しなければならない。ユニットスタンプは混合行内の同一フィールドの異なる意味を区別できない。

通常の互換オープンと読み取りは保存値を再スタンプも正規化もしない。ユニット全体の JSON は現在のスタンプと変更データを 1 回のアトミックなファイル置換で公開する。SQLite は最初に更新されたユニットスタンプと変更を 1 つのトランザクションでコミットする。バッチ削除は常に 1 つのトランザクションを使う。存在しないレコードや空のバッチの削除はスタンプを進めない。スタンプは受け入れられたフォーマットを識別し、値の正規化の完了を識別しない。読み取り側は混合世代を受け入れなければならない。ユニット全体のスタンプを持つバックエンドは、サポートされないユニットバージョンを拒否する。

公開前のシリアライズ失敗または SQLite のロールバック成功は、以前のデータとスタンプを保持する。JSON 置換後の I/O 失敗、または SQLite ロールバック自体の失敗は、拒否にもかかわらず保存結果を不明のままにすることがある。バックエンドとそのドメインの使用を止めて閉じ、再オープンして保存状態を検査してから再試行するかどうかを決める。バックエンドの README がこれらの失敗ケースを定義する。

JSON の `per-record` レイアウトは各ドキュメントを独立に検査しスタンプする。サポートされないドキュメントは存在しないものとして読まれる。既存のユニット全体ブートストラップは読み取り専用ローディングの例外である: 受け入れられた歴史的値を変換せずに現在のスタンプを持つドキュメントへコピーする。既存の宣言済みテーブルディレクトリ（空でも）または宣言済みグローバルドキュメントはブートストラップを抑制する。SQLite はレイアウト要求に関係なくユニット全体のバージョン検査を保持する。レイアウトと復旧動作については [JSON バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-json/README.md) と [SQLite バックエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-sqlite/README.md) を参照。

互換な後継では、`version` を上げ、引き続き読めるすべてのスタンプを `compatibleVersions` に保持し、同じスキーマのデコード/デフォルトを拡張する。コンシューマーは選択したレコードを書き換えるときに値を正規化する。Storage はアプリケーションのレコードバージョンを割り当てない。歴史的およびスキップされたアップグレード、混合値、書き込みなしの読み取り、失敗ロールバック、再起動、他ユニットの分離、未知バージョンの拒否を検証する。テーブル削除とレイアウト変更には独自の保持とアップグレードポリシーが必要であり、互換スタンプだけではどちらも定義されない。

v2 ユニットは未変更の v1 値を含んでよい。スタンプ 2 を受け入れる v3 リーダーはそれらの値を依然としてデコードしなければならない。スタンプ 1 の受け入れも、v2 を完全にスキップしたユーザーには必要である。古いデコードを削除するには、現在のユニットスタンプ下にあるものを含め、影響を受けるすべてのレコードを書き換えて検証する必要がある。互換性のない変更には、新しいスキーマで開く前に明示的で復旧可能なアップグレードが必要である。ストレージドメインには値移行ランナーがない。

古いリーダーへの復帰が必要な場合は、新しいフォーマットを書く前に一貫したバックアップを取る: そのリーダーは最初の変更後に新しいスタンプを拒否できる。保存スタンプを手作業で下げても値の変換は逆転しない。ドメインの互換性は SQLite の物理スキーマや JSON エンベロープのフォーマットを移行せず、バックエンドルートの変更は保存データを転送しない。セッションフォーマットの移行と SQLite の物理レイアウトバージョニングには別の所有者があり、ドメイン値を移行しない。

## オープン済みドメイン

```ts type-equiv
/** One open domain, typed by its spec. */
interface Domain<S extends DomainSpec> {
  /** Domain name from the spec. */
  readonly name: string
  /** Global singleton handle; a spec without `global` has no usable handle (`never`). */
  readonly global: DomainGlobalHandleOf<S>
  /**
   * Resolve one declared table handle. Handles are stable — repeated calls
   * return the same instance.
   * @param name - Declared table name.
   * @returns the typed table handle.
   */
  table<N extends keyof S['tables'] & string>(name: N): KvTable<TableKeyOf<S, N>, TableValueOf<S, N>>

  /**
   * Close this domain: reject new writes immediately, drain already-queued
   * writes (their events still emit), release the backend unit, then free
   * the domain name for a later open. Idempotent — repeated calls share one
   * teardown. The consumer owns this call (typically as its own `ctx.effect`
   * disposer); the facility closes any domain left open when it unmounts.
   * @returns resolution after the unit is released.
   */
  close(): Promise<void>
}
```

読み取りは権威あるインメモリ状態から同期的に行われる: `KvTable` は `get`/`entries`/`keys`/`size` を公開し（キュー済み書き込みが着地する間も安定したスナップショットイテレーター）、グローバルハンドルの `get()` は最初の `set` が媒体上にスロットを実体化するまで spec の `initial` を提供する。すべての書き込み — `put`、`delete`、`deleteMany`、`update`、`global.set` — はドメインごとの 1 つのチェーンにキューされ、最初にバックエンドの永続性に達し、次にメモリを変更し、次に `domain/changed` を emit する。拒否されたバックエンド書き込みはドメイン変更を公開しない。未知の永続結果はそのスナップショットをさらに使う前に再オープンを要求する。`unknown-state` を報告するバックエンドは代わりにドメインを隔離し、その読み取りと後続の書き込みは `close()` が解放するまで拒否する。`update(key, fn)` はチェーンスロットでのアトミックな読み取り-変更-書き込みである（欠落キーは `missing-key` を拒否する）。存在しないキーの `delete` は書き込みもイベントもなく `false` で解決する。`deleteMany(keys)` は存在するキーを 1 つのチェーンスロットで 1 つのバックエンドバッチとして削除し、削除数を解決し、削除されたキーごとに 1 つの `domain/changed` を emit する。拒否されたバックエンド書き込みはすべてのレコードをそのまま残し、何も emit しない。`unknown-state` の拒否はドメインを隔離する。返されるレコードはコピーではなく保存されたオブジェクトそのものである — `put`/`update` で置換し、決してインプレースで変更しない。

## ドメインファシリティ: `ctx.storageDomain`

`DomainFacility`（[シグネチャ](#ctxstoragedomain--domainfacility)）はルーティングされたバックエンド上で宣言済みドメインを開く。ルーティングはドメインプラグインの設定であり、ハブの設定ではない: `backend` は必須のデフォルトルートを名付け、`routes` はドメイン名ごとにそれを上書きする。`open(spec)` は厳密な順序で実行され、各ステップが呼び出し全体を失敗させる: すでに開いているかまだ閉じている途中の名前を拒否し（`already-open`）、ルートを解決し（`backend-not-found`）、バックエンドの `kv` ファセットを要求し（`facet-unsupported`）、ユニットを開き（バックエンドの `version-mismatch`/`malformed-medium` はそのまま通過する）、すべての保存レコードとグローバルを spec の zod スキーマに対して検証する（`invalid-record` と問題のあるテーブルおよびキー）。呼び出し元は返されたハンドルを所有し、`Domain.close()` で解放する。プラグインのアンマウント時にまだ開いているドメインはファシリティによって閉じられ、閉じたドメインの名前はティアダウンが完全に完了した後にのみ再オープン用に解放される。`get(name)` はすべての型付きハンドルの背後にあるパッケージプライベートな `DomainImpl` ランタイムへの型なし診断ルックアップである。`closeAll()` はアンマウント経路である。

## 変更イベント: `domain/changed`

すべての永続書き込みは、バックエンドが永続性を確認した厳密に後に、ドメインの書き込みチェーン順で 1 つのイベントを emit する（[イベントエントリー](#domainchanged--emit)）:

```ts type-equiv
/** Shared location fields of one durable domain change. */
interface DomainChangedBase {
  /** Owning domain name. */
  readonly domain: string
  /** Table name; `''` for a global-singleton write. */
  readonly table: string
  /** Record key; `''` for a global-singleton write. */
  readonly key: string
}
```

```ts type-equiv
/** One durable domain change; a closed union — switch on `operation`. */
type DomainChanged = DomainChangedPut | DomainChangedDeleted
```

`put`（挿入、上書き、グローバル書き込み）は新しいスナップショットを `value` で運ぶ — 古い値ではない。差分を取るコンシューマーは自身の以前のスナップショットを保持する。`deleted` は値を持たないトゥームストーンである。イベントは通知でありトランザクション参加者ではない: emit 時点でコミットポイントは過ぎているため、同期的にスローするリスナーはすでに永続化された書き込みを拒否するのではなく、警告をログに記録して封じ込められ、emit された値は emit 時点のインメモリ状態と等しい。イベントはインプロセス専用である。クロスプロセスの変更プッシュは記録された制限である（[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-domain/README.md)）。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxstorage--storage"></a>

### `ctx.storage` — `Storage`

ストレージハブサービス。バックエンドは `backend` の下に登録し、データフォームは自身の `StorageForms` キーの下にマウントされ、`ctx.storage.<form>` として到達される。

```ts cordis-catalog
/**
 * Mount a data-form facility on the hub. Mounting is an effect: the
 * returned disposer unmounts the form.
 * @param form - Form key declared in {@link StorageForms}.
 * @param facility - The facility instance to expose.
 * @returns the disposer that unmounts the form.
 */
mount<K extends keyof StorageForms>(form: K, facility: StorageForms[K]): () => void

/**
 * Resolve a mounted data form.
 * @param form - Form key declared in {@link StorageForms}.
 * @returns the mounted facility.
 */
form<K extends keyof StorageForms>(form: K): StorageForms[K]
```

ソース: [`packages/storage/storage/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage/src/index.ts)

<a id="ctxstoragedomain--domainfacility"></a>

### `ctx.storageDomain` — `DomainFacility`

マウントされたドメインファシリティ。ルーティングされたバックエンド上で宣言済みドメインを開く。1 つのファシリティインスタンスがオープンドメインテーブルを所有し、ドメイン名ごとの単一オープンを強制する。

```ts cordis-catalog
/**
 * Open one declared domain. Steps, each failing the whole call: reject a
 * name that is already open (`already-open`); resolve the backend route
 * (`backend-not-found` passes through from the hub); require its `kv` facet
 * (`facet-unsupported`); open the unit projected from the spec (backend
 * `version-mismatch`/`malformed-medium` pass through); load and validate
 * every stored record against the spec's zod schemas (`invalid-record`
 * with the offending table and key — unless the spec declares
 * `invalidRecords: 'backup-and-skip'` and the unit can move documents aside, in
 * which case the failing record is backed up, logged, and skipped);
 * construct the domain.
 *
 * Lifecycle: the CALLER owns the returned handle and closes it via
 * `Domain.close()` (typically as its own `ctx.effect` disposer) — the
 * facility does not tie the domain to any consumer fiber. Domains still
 * open when the facility unmounts are closed by the plugin disposer.
 * @param spec - The domain declaration, typically from `defineDomain`.
 * @returns the opened domain handle, typed by the spec.
 */
async open<S extends DomainSpec>(spec: S): Promise<Domain<S>>

/**
 * Look up an open domain by name, untyped. Diagnostic surface; typed
 * consumers hold the handle returned by {@link open}.
 * @param name - Domain name.
 * @returns the open domain runtime, or `undefined` when not open.
 */
get(name: string): DomainImpl | undefined

/**
 * Close every domain still open on this facility. The unmount path for
 * consumers that never called `Domain.close()` themselves; closing is
 * idempotent, so double-closing an already-closed domain is harmless.
 * @returns resolution after every unit is released.
 */
async closeAll(): Promise<void>
```

ソース: [`packages/storage/storage-domain/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-domain/src/index.ts)

<a id="domain-events"></a>

### `domain/*` イベント

<a id="domainchanged--emit"></a>

#### `domain/changed` — emit

ドメインレコードまたはグローバルシングルトンが変更された。バックエンドが永続性を確認した厳密に後に、書き込みごとに 1 回 emit される。1 つのドメインのイベントはその書き込みチェーン順で到着する。

```ts cordis-catalog
/**
 * A domain record or the global singleton changed, emitted once per write
 * strictly after the backend acknowledged durability. Events of one
 * domain arrive in its write-chain order.
 * @param change - domain, table (`''` for global), key (`''` for global),
 * operation discriminant, and on `put` the new snapshot.
 * @mode emit
 */
'domain/changed'(change: DomainChanged): void
```

ソース: [`packages/storage/storage-domain/src/events.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/storage/storage-domain/src/events.ts)
<!-- END GENERATED cordis-surface -->
