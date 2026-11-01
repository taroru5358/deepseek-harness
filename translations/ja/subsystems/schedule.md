# ホスト全体のスケジュール

日本語 | [English](../../../docs/subsystems/schedule.md) | [中文](../../../docs/subsystems/schedule.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/schedule.md](../../../docs/subsystems/schedule.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Schedule は Session のアクティブ化とは独立に Host タスクを保存する。1 つの管理 API が id でタスクを扱い、1 つのエバリュエーターが 7 つの権威的なタイミング種別（`after`、`at`、`every`、`daily`、`weekly`、`cron`、`rrule`）を処理する。ターゲットは既存の Session を再利用するか、1 つの Session を作成して再利用するか、配信ごとに Session を作成する。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/README.md) がコンポジションとリマインダーのフレーミングを所有し、[`types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/src/types.ts) が完全な値型とリクエスト型を所有する。

<a id="durable-tasks"></a>
## 永続タスク

プラグイン README が現在の [v2 タスク・スキーマ、v1 デフォルト、永続化 JSON の例](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/README.md#storage-format) を所有する。ドメインはその名前、ルーティング、ユニット全体のレイアウトを保つ。7 つのタイミング種別はすべて権威的なままであり、正規化はコミット済みの発生を保存し、Cron を RRULE に変換しない。

不正なストレージ、無効な行、一致しないタスク・キー、サポートされないバージョンは、タスクがアクティブ化する前に拒否する。開くことの失敗は媒体を変更しない。ファイルがない場合は空で始まる。存在しないターゲット Session は管理画面に表示されたままである。タスクの読み取りがその Session をアクティブ化することは決してない。

<a id="storage-versioning"></a>
## ストレージのバージョニング

README は[タスクがいつアップグレードされるか](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/README.md#storage-format)、SQLite の変わらない物理スキーマ、変更コスト、将来の互換 v3 を定義する。`compatibleVersions` は現在のデコードのために古いユニット・スタンプを受け入れる。通常の編集と配信は現在の行を永続化する。読み取りはストレージを変更しない。共有の[ドメイン・バージョン・ポリシー](storage.md#domain-versioning)が JSON と SQLite の両方に適用される。

<a id="recurrence-rules"></a>
## 繰り返しルール

[`dsh-util-time`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/time/README.md) は再利用可能な `Cron` と `RRule` のパースと境界付きの発生演算を所有する。パース済みの両方の値は、権威的な式とゾーン、`next(after)`、`resolve(scheduledAt, acceptedAt)` を公開する。パーサーは受け入れ値または拒否理由を返す。Schedule は理由を自身の入力エラーにマップし、レコード検証、受け入れ制限、永続化、ディスパッチを所有する。

保存されるすべての `scheduledAt` は、ミリ秒精度で年が 0001 から 9999 の、正規の RFC 3339 UTC インスタントである。デコードは現在のゾーン・データからそれを再計算しない。最新のみのキャッチアップはそのコミット済みの下限を保存し、最新の期限到来分を 1 回配信し、最初の未来の発生をコミットする。枯渇するとタスクは `ended` として保存されたままになる。

| 種別 | 権威的なタイミング |
|---|---|
| `after` | 正の整数 `afterSeconds` と、コミット済みの一回限りの UTC ターゲット。 |
| `at` | コミット済みの一回限りの UTC ターゲット。入力は明示的オフセット付きインスタント、またはローカル日付・時刻・明示的ゾーンを受け付ける。 |
| `every` | 少なくとも 60 の整数 `everySeconds`。発生はコミット済みの間隔アンカーに整列したままである。 |
| `daily` | `HH:mm:ss.SSS` 形式のローカル `time` と明示的な `timeZone`。 |
| `weekly` | 日次フィールドに加え、空でなく一意で昇順の ISO `weekdays` セット（月曜 1 から日曜 7）。 |
| `cron` | 正規の 5 フィールド `expression` と明示的な `timeZone`。 |
| `rrule` | `rrule` 内の 1 つの正規化された `DTSTART` 行と 1 つの `RRULE` 行。`DTSTART` がアンカーとゾーンを供給する。 |

ローカル・カレンダー・ルールは存在しない壁時計時刻をスキップし、重複の場合は早い方のインスタントを選ぶ。固定間隔は経過時間の意味を保つ。一回限りの遅延は 1 分未満でもよいが、タスクごとの配信下限は最初の配信後も適用される。

<a id="absolute-time-input"></a>
## 絶対時刻の入力

`at` セレクターは、明示的な `Z` または数値オフセット付きの RFC 3339 インスタント、あるいは `{ date, time, time_zone }` を受け付ける。ローカル日付は `YYYY-MM-DD`、ローカル時刻は `HH:mm:ss` に任意で 1〜3 桁の小数部を付けたものを使う。曖昧または存在しないカレンダー値は、共有の時間ユーティリティを通じて検証される。新しい絶対ターゲットは厳密に未来であり、表現可能でなければならない。

<a id="daily-wall-clock-input"></a>
## 日次・週次の入力

日次入力は `{ time, time_zone }` を供給し、週次は明示的な `weekdays` を追加する。作成は厳密に未来の最初の実際の発生を選ぶ。キャッチアップは最新の実際の期限到来発生を選び、後に一致するローカル日付へ進む。等価な編集は、ミリ秒を含め、保存済みの IANA エイリアスと固定された UTC ターゲットを保持する。

<a id="fixed-rate-input-and-catch-up"></a>
## 固定レートの入力とキャッチアップ

every 入力は `every_seconds` を供給する。新しい間隔は受け入れ時刻から始まり、変更された間隔は編集の受け入れ時に新しいアンカーを確立する。変更されていない間隔、または名前／prompt の編集は古いアンカーを保つ。ダウンタイムは、各発生を再生する代わりに逃した間隔を合流させる。

<a id="cron-wall-clock-input"></a>
## Cron 入力

Cron 入力は `{ expression, time_zone }` を供給する。その 5 つの数値フィールドは分、時、日、月、曜日である。サポートされる原子は `*`、値、両端を含む範囲、スターまたは範囲のステップ、カンマ区切りリストである。日曜は 0 または 7 を受け付け、0 に正規化される。名前、マクロ、秒フィールド、値の後にステップが続くものは拒否される。

Cron は Vixie の日マッチングを保つ。両方の日フィールドが制限されている場合、day-of-month **または** day-of-week の一致でよい。いずれかのフィールドが `*`（ステップ付きスターを含む）で始まる場合、両方の一致セットが一致しなければならない。正規化はその区別を保つ。探索は 1 つの 400 年グレゴリオ暦周期と 4 桁の UTC 範囲で制限される。表現可能な未来の発生がないルールは、アーム時に拒否される。

Cron レコードは決して RRULE に変換されない。保存された式とゾーンは、読み取り、記述的な編集、再ターゲット、再起動を通じて権威的なままである。Custom エディターは 5 フィールドの Cron 式を受け付け、その明示的なゾーン・セレクターを公開する。

<a id="rrule-input"></a>
## RRULE 入力

RRULE は、共有の [`rrule.ts` パーサー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/time/src/rrule.ts) で定義される境界付きの RFC 5545 サブセットを受け付ける。入力はちょうど 2 つの空でない内容行、`DTSTART` とそれに続く `RRULE` を持つ。開始は UTC の `Z` または明示的な IANA `TZID` を伴う実際の `YYYYMMDDTHHMMSS` 日時である。フローティングなローカル時刻、数値オフセット、日付のみの値、余分な内容行は拒否される。エンジンは `rrule` 2.8.1 を使い、`MINUTELY`、`HOURLY`、`DAILY`、`WEEKLY`、`MONTHLY`、`YEARLY` をサポートする。一回限りは `COUNT=1` を使う。

受け付けられるのは `FREQ`、`INTERVAL`、`COUNT`、`UNTIL`、`WKST`、`BYSECOND`、`BYMINUTE`、`BYHOUR`、`BYDAY`、`BYMONTHDAY`、`BYYEARDAY`、`BYWEEKNO`、`BYMONTH`、`BYSETPOS` だけであり、名前とリスト値は一意でなければならない。`INTERVAL` は正の安全な整数である。`WKST` は `MO` から `SU` までの曜日で、デフォルトは `MO` である。数値リストはパーサーの範囲に収まらなければならない。分／時のルールは `BY` パーツを受け付けない。`BYMONTHDAY` と序数の `BYDAY` は Monthly または Yearly を要求し、`BYYEARDAY` と `BYWEEKNO` は Yearly を要求する。序数の `BYDAY` は `BYWEEKNO` と組み合わせられず、`BYSETPOS` は別の `BY` パーツを要求する。

検証は列挙に先行する。`SECONDLY`、複数の `BYSECOND` 値、周波数期間あたり 10,080 を超える可能な候補は拒否される。`COUNT` は 1 から 1,000 で、`UNTIL` と共存できない。`UNTIL` は `Z` で終わる絶対 UTC 日時である。開始と制限は 0100 から 9999 の年を使う。受け入れられるセット位置は最大 31 で、それぞれ −366 から −1、または 1 から 366 であり、境界付きの期間内でなければならない。より厳格な新規ルール作成プロファイルは、期間に候補が最大 1 つしかない場合に `BYSETPOS` も拒否する。サポートされる未変更ルールのデコードと再開は、その通常のエバリュエーターを使う。

存在しないローカルの `DTSTART` は拒否される。後で欠落するローカル発生は `COUNT` を消費せずにスキップされ、重複は早い方のインスタントを選ぶ。`COUNT` は各実際の発生を 1 回ずつ数え、正と負のセット位置が同じ発生を選ぶ場合も同様である。

カスタム RRULE テキストはその `DTSTART` ゾーンを保持する。モデル向けツールは引き続き RRULE を教え、受け付ける。共有管理 API は各タイミング種別を明示的に受け付ける。

<a id="delivery-targets"></a>
## 配信ターゲット

| ターゲット | 配信挙動 |
|---|---|
| `session` | 1 つの既存 Session に配信する。アーム時に管理 FIFO 内で、それが存在し、アーカイブされておらず、委譲された子でもないことを確認する。配信はアーカイブ済みの宛先を再確認する。 |
| `new_once` | 最初の配信前に Session id を予約し、必要になったときにその正確な Session を作成してから `session` を保存して再利用する。再試行と再起動は予約を保持する。 |
| `new` | 配信ごとに Session を作成する。作成前に `pendingDelivery.targetSessionId` を永続化するため、失敗した試行は同じ Session で再試行する。 |

省略された作成ターゲットは `new_once` に既定する。`new` または `new_once` ターゲットの追加または置換には、埋め込み側の Session 作成プロバイダーが必要である。それを持たない SDK コンポジションは、保存前にそうしたリクエストを `invalid_rule` で拒否する。既存の `session` ターゲットは引き続き利用できる。作成値 `cwd` と `agentPreset` は任意で、Host のデフォルトを使う。作成された Session は、そのディレクトリに登録された Workspace があればそれに参加し、無題の間はタスク・タイトルを取る。`new` はルールのゾーンで作成分を追加する。参加または名前変更の失敗はログに記録され、配信をブロックしない。

<a id="task-management"></a>
## タスク管理

すべての呼び出し元は同じ `ScheduleCreateRequest`、`ScheduleCatalogEntry`、`ScheduleTiming`、`ScheduleEditRequest` を使う。ブラウザ・セーフな型は `@deepseek-ai/dsh-schedule/client` から来る。Session に束縛された管理サーフェスも、第 2 の型エイリアス群も存在しない。

| 操作 | 挙動 |
|---|---|
| `create(request, actorSessionId?, signal?)` | id を割り当て、タイトル、prompt、型付き `timing`、任意のターゲットを保存する。 |
| `list(request?)` | ページを読む。デフォルト 100、最大 500、任意の `active` フィルターと非負の `offset`。 |
| `catalog()` / `view({ id })` | ページ制限なしですべてのタスクを読むか、1 つのタスクを読む。 |
| `history({ id, limit, before? })` | Session をアクティブ化せずに保存済み配信を読む。limit は 1 から 100。 |
| `edit(request, actorSessionId?, signal?)` | `update` パッチまたは `delete` を id でアトミックに適用する。 |
| `deleteInactive()` | `ended` のタスクだけを 1 バッチで削除し、アクティブと一時停止のタスクを保持し、`{ deleted }` を返す。空集合は書き込みを行わない。 |
| `run({ id }, signal?)` | スケジュールされた発生やライフサイクルを変えずに、通常の受け入れを通じて今すぐ配信する。 |
| `deliveryLimits()` | `new` ターゲットに対して設定された最小間隔を読む。 |

Remote ルートは `create`、`list`、`catalog`、`view`、`history`、`edit`、`deleteInactive`、`run`、`deliveryLimits` を公開し、トランスポート・ラッパーは同じ管理メソッドへ委譲する。Remote の検証失敗は Schedule の入力コードを伴う `schedule/input-invalid` を使う。ネイティブ呼び出し元は `ScheduleInputError` を受け取る。

空でない `deleteInactive` バッチが成功すると、リーダーが更新するための `schedule/changed` を発行する。Agent 通知はキューに入れない。拒否されたバッチもそのイベントを発行し、リーダーが古い状態を捨てられるようにする。通常の失敗はすべての行を保存する。`unknown-state` の失敗は行を削除している可能性があり、再び開くまでドメインを隔離する。失敗した呼び出しは削除件数を報告する代わりに拒否する。

更新はタイトル、prompt、型付きタイミング、ターゲット、またはアクティビティを置換してよい。省略されたフィールドは値を保つ。変更されたルールまたは明示的な再開は、FIFO 受け入れ時に未来の発生を計算する。等価なタイミングと記述的な編集はコミット済みターゲットを保つ。配信済みの一回限りは、再開するために新しい未来のタイミングを必要とする。再ターゲットはそのタイミング種別の変換を必要としない。

`expectedUpdatedAt` は任意の楽観的比較である。文字列は保存済みリビジョンと比較し、明示的な `null` はリビジョンの不在と比較する。不一致は `schedule_conflict` を返す。リビジョンのない行の編集はリビジョンを作成し、以降のリビジョンは時計が巻き戻っても単調に進む。キャンセルは書き込みが始まる前に確認され、開始済みの永続書き込みはロールバックされない。

<a id="timing-edits"></a>
## タイミング編集

`timing` の判別子は、それに一致するペイロードを正確に選ぶ。たとえば `{ kind: "cron", cron: { expression: "0 9 * * 1-5", time_zone: "Asia/Shanghai" } }` や `{ kind: "rrule", rrule: "DTSTART:20990101T090000Z\nRRULE:FREQ=DAILY" }` である。他のセレクターは `after_seconds`、`at`、`every_seconds`、`daily`、`weekly` を使う。無効な組み合わせは永続化前に拒否される。UI は型付きの Cron/RRULE 入力をこの同じ API で保存し、触れていないタイミングを保持する。

<a id="saved-delivery-records"></a>
## 保存済み配信レコード

レシートは `scheduledAt`、`deliveredAt`、`messageId`、`targetSessionId` を記録する。これはモデルの完了ではなく、永続的な Session インボックス書き込みを承認するものである。保存済みレコードは、不変の送信済み `prompt` を持つこともある。ターゲット id のない古いレシートは、再ターゲット編集の前に解決済みの再利用可能な宛先を使うため、後のターゲットがその履歴を書き換えることはできない。明示的な宛先も既知の再利用可能ターゲットも持たないレシートはデコードを拒否する。新しいデフォルト予約は歴史的な宛先を確立できない。

新しいタスクは既知の空の履歴で始まる。`deliveryHistory` のない古い行は、存在すれば最新のレシートを公開し、`earlierRecordsUnavailable: true` を設定する。失われた履歴や prompt スナップショットをでっち上げることはない。省略された `earlierRecordsPruned` は false として読まれる。重複するメッセージ id、または最新の保存済みレコードと `lastDelivery` の不一致はデコードを拒否する。

履歴は時計の巻き戻しにもかかわらず追記順に従う。承認は同じタスク・コミットで追記とプルーニングを行い、デフォルトで最大 200 レコードと 30 日を保持しつつ、最新のレシートを残す。クエリ・ページは新しい順で、排他的なメッセージ id カーソルを使う。ハード削除はタスクとその履歴を削除するが、すでにキューに入った Session メッセージは削除しない。

<a id="host-delivery"></a>
## Host 配信

1 つの Host タイマーがアクティブ・タスクを駆動する。runtime と管理の書き込みは 1 つの FIFO を共有する。既存および予約済みの Session は `schedule/resolve-agent` を通じて解決されるため、アクティブ化は埋め込み側が所有し、注入された Web コントローラーはフォールバックである。Session 再利用タスクは、ターゲット Agent がアイドルになるまで実行スロットを保持し、その間の発生は合流する。`new` ターゲットは実行を並行させられる。同じ既存 Session をターゲットとする期限到来の繰り返しレコードは 1 つのメッセージを共有し、`after` と `at` は別々に配信する。

時間制限付きおよび手動の配信は、タスクごとの最小間隔 60 秒に従う。`newSessionIntervalSeconds` は `new` ターゲットに対してそれを最大 1 週間まで引き上げられる。下限は、保存済みの後続が間隔を保つ場合は最後の発生を、そうでなければ最後の承認を使うため、配信遅延がルールのアンカーを動かすことはない。読み取りは `executionActive` と `deliveryFloorEndsAt` を該当する間だけ公開する。

Session インボックスの永続化はタスク・レシートの書き込みに先行するため、ストア間のクラッシュは配信を繰り返す可能性がある。Host は実行中でなければならない。Session をターゲットとするアクティブ・タスクはアーカイブ受け入れアクティビティに寄与し、アーカイブ停止はそれらのアクティブ・タスクを削除する。

コミットされた管理変更の後、Schedule はアクターを除く関与するロード済み Agent それぞれにベストエフォートの通知をキューに入れる。通知は turn を開かずに次のモデル・ステップに参加し、その Session I/O は管理／配信キューを保持しない。配信ごとに新しい Session は関与メタデータに蓄積されない。

<a id="historical-session-changes"></a>
## 歴史的 Session 変更

歴史的なバージョン 1 の `schedule/change` イベントは、凍結された `LegacyScheduleRecord` 型を保つ。`after`、`at`、`every` であり、古い任意タイトルを含む。それらは Schedule インタープリター、警告フック、インポーター、タイマーを持たない、記録済みの Session データのままである。`schedule.json` の既存 Host タスクは引き続き通常のデコーダーとエバリュエーターを使う。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxschedule--scheduleservice"></a>

### `ctx.schedule` — `ScheduleService`

共有管理サービス。読み取りと管理書き込みは Session をアクティブ化せず、配信だけが Session を作成または再開する。

`sessionPersistence` はヘッダーのみのターゲット検証と、`ctx.sessions.flush()` を通じて永続インボックス配信を承認する `session/flush` リスナーを提供する。

```ts cordis-catalog
/** Read effective delivery limits without loading or activating a task.
 * @returns The configured minimum spacing between deliveries that each create a Session, in seconds.
 */
@Remote('deliveryLimits') deliveryLimits(): { readonly newSessionIntervalSeconds: number }

/**
 * Create one task with a Host-generated identity for a human Remote caller.
 * @param request - Title, prompt, recurrence, and delivery destination.
 * @param signal - Cancellation checked before the durable write begins, including after FIFO waits.
 * @returns The stored task. Cancellation does not roll back an in-flight write.
 */
@Remote('create') async remoteCreate(request: ScheduleCreateRequest, signal?: AbortSignal): Promise<ScheduleCatalogEntry>

/**
 * Create one task and record the acting Agent's Session as involved; the Remote form omits that Session.
 *
 * The record is built from the clock reading taken before the request joins the serialized queue. A `session`
 * target must exist and must not belong to a delegated child; a `new_once` target reserves its Session identity.
 * @param request - Title, prompt, recurrence, and delivery destination.
 * @param actorSessionId - Session of the acting Agent, or undefined for a human caller.
 * @param signal - Cancellation checked before the durable write begins, including after FIFO waits.
 * @returns The stored task. Cancellation does not roll back an in-flight write.
 */
async create( request: ScheduleCreateRequest, actorSessionId?: SessionId, signal?: AbortSignal, ): Promise<ScheduleCatalogEntry>

/**
 * Read one page of Host tasks without activating target Sessions.
 * @param request - Optional activity filter, page size, and offset.
 * @returns The requested page and total matching task count.
 */
async list(request: ScheduleListRequest = {}): Promise<ScheduleListResult>

/**
 * Remote form of the shared Host-wide list.
 * @param request - Optional activity filter, page size, and offset.
 * @returns The requested page and total matching task count.
 */
@Remote('list') async remoteList(request: ScheduleListRequest): Promise<ScheduleListResult>

/**
 * Read every Host task without a page bound or Session activation.
 * @returns All tasks, active first, with target and execution metadata.
 */
@Remote('catalog') async catalog(): Promise<ScheduleCatalogEntry[]>

/**
 * Read one task without activating its target Sessions.
 * @param request - Task identity.
 * @returns The task, or a non-mutating miss.
 */
@Remote('view') async view(request: ScheduleViewRequest): Promise<ScheduleViewResult>

/**
 * Read one page of saved deliveries without activating or reading a target Session.
 * @param request - Task identity, explicit page size, and optional exclusive cursor.
 * @returns Newest-first saved deliveries, or a task/cursor lookup failure.
 */
async history(request: ScheduleDeliveryHistoryRequest): Promise<ScheduleDeliveryHistoryResult>

/**
 * Remote form of the target-based {@link ScheduleService.history}; an invalid limit rejects with
 * `schedule/input-invalid`.
 * @param request - Task identity, explicit limit, and optional exclusive message cursor.
 * @returns Newest-first deliveries in append order, or a task/cursor lookup failure.
 */
@Remote('history') async remoteHistory(request: ScheduleDeliveryHistoryRequest): Promise<ScheduleDeliveryHistoryResult>

/**
 * Deliver one task now without consuming or moving its scheduled occurrence; an archived destination Session rejects
 * with `schedule/target-archived`.
 * @param request - Task identity.
 * @param signal - Cancellation checked after entering the delivery transaction and before delivery.
 * @returns Task state after Session persistence acknowledged the delivery, `schedule_execution_pending` while
 * another execution of the task is queued, running, or inside its delivery floor, or a non-mutating miss.
 */
@Remote('run') async run(request: ScheduleRunRequest, signal?: AbortSignal): Promise<ScheduleRunResult>

/**
 * Delete every ended Host task, leaving active tasks armed.
 *
 * Ended tasks never schedule another delivery, so the removal changes no pending
 * target and no timer is recomputed. Rows are enumerated and removed in one FIFO
 * slot, so a delivery ordered behind this call ends its task after the
 * enumeration rather than during it. The batch is one domain write: a normal
 * failure rejects with every listed row still stored, while a backend
 * `unknown-state` failure may have removed rows and quarantines the Host
 * domain, so later schedule operations reject until the domain is reopened.
 * A successful nonempty batch emits `schedule/changed` without Agent notices.
 * A rejected batch announces `schedule/changed` because it does not establish
 * which rows landed, and clients then re-read the catalog.
 * @returns Count of removed rows; zero when the Host held no ended task.
 */
@Remote('deleteInactive') async deleteInactive(): Promise<ScheduleDeleteInactiveResult>

/**
 * Update or delete one task for a human Remote caller; an invalid patch rejects with `schedule/input-invalid`.
 * @param request - Task identity and one update or delete operation.
 * @param signal - Cancellation checked before the durable write begins, including after FIFO waits.
 * @returns The updated task, the deletion outcome, or a non-mutating miss or conflict.
 */
@Remote('edit') async remoteEdit(request: ScheduleEditRequest, signal?: AbortSignal): Promise<ScheduleEditResult>

/**
 * Update or delete one task and record the acting Agent's Session as involved; the Remote form omits that Session.
 *
 * An update applies every patch field in one write and recomputes the next occurrence when the rule changes or
 * the task resumes. A supplied `expectedUpdatedAt` that differs from the stored value returns `schedule_conflict`.
 * @param request - Task identity and one update or delete operation.
 * @param actorSessionId - Session of the acting Agent, or undefined for a human caller.
 * @param signal - Cancellation checked before the durable write begins, including after FIFO waits.
 * @returns The updated task, the deletion outcome, or a non-mutating miss or conflict.
 */
async edit( request: ScheduleEditRequest, actorSessionId?: SessionId, signal?: AbortSignal, ): Promise<ScheduleEditResult>
```

型: [SessionId](core.md)

ソース: [`packages/schedule/schedule/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/src/index.ts)

<a id="schedule-events"></a>

### `schedule/*` イベント

<a id="schedulechanged--emit"></a>

#### `schedule/changed` — emit

永続タスク集合が変更された。クライアントはグローバル・タスク・カタログと Session アクティブ・カタログを再取得する。

```ts cordis-catalog
/** Durable task set changed; clients refetch global task and Session-active catalogs.
 * @mode emit
 */
'schedule/changed'(): void
```

ソース: [`packages/schedule/schedule/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/src/types.ts)

<a id="scheduleresolve-agent--waterfall"></a>

#### `schedule/resolve-agent` — waterfall

埋め込み側の所有コントローラーを通じてリマインダーの Session を復元する。Web Session コントローラーを使うには next() を呼ぶ。

```ts cordis-catalog
/**
 * Restore a reminder's Session through its embedding's owning controller.
 * Call next() to use the Web Session controller.
 * @param sessionId - Original Session receiving the reminder.
 * @mode waterfall
 */
'schedule/resolve-agent'(sessionId: SessionId, next: () => Promise<Agent>): Promise<Agent>
```

型: [Agent](core.md) · [SessionId](core.md)

ソース: [`packages/schedule/schedule/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/src/index.ts)
<!-- END GENERATED cordis-surface -->
