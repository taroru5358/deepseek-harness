# スピル・ストレージ

日本語 | [English](../../../docs/subsystems/spill.md) | [中文](../../../docs/subsystems/spill.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/spill.md](../../../docs/subsystems/spill.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

スピル・ストレージの[ケイパビリティ seam](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.md)は、呼び出し元が提供したテキストを永続化し、取得のガイダンスを伴うモデル向けロケーターを返す。その Service Definition は [dsh-spill](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/spill/spill)（`ctx.spillStore`）であり、ローカルの Service Provider は [dsh-spill-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/spill/spill-local) である。コンシューマーには[ツール結果ポリシー](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/spill/spill-policy)と[セッション参照](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/session-reference/README.md)がある。スピルは任意であり、[agent ループの背骨](core.md)の一部ではない。プレビューとスピルの判断はコンシューマーが所有し、ストレージは渡されたテキストをそのまま保存する。

ソース: [`packages/spill/spill/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill/src/types.ts)

## 保存要求

`saveText` が唯一のサービス操作である。`content` をそのまま永続化し、不透明なロケーター、バックエンドが提供する取得ヒント、正確なバイト数を返す。要求は保存時のストレージ名前空間（`owner`）、記述的なプロデューサー詳細（`source`、アクセス制御では決してない）、そしてバックエンドが命名ヒントとして使ってよい `suggestedName` を運ぶ。`suggestedName` はパスではない。ツールのソースは実際のツール呼び出しを識別し、セッション参照のソースはキャプチャされたソース・セッションを識別する一方、その owner はコンテキストを受け取るターゲット・セッションである。

```ts type-equiv
/** One request to persist text to a spill artifact. */
interface SaveTextSpill {
  owner: SpillOwner
  source: SpillSource
  /**
   * A caller-suggested base name (e.g. `web_fetch.txt`). The backend sanitizes
   * it to a single safe path segment before use — it is a hint, never a path.
   */
  suggestedName: string
  /** The full text to persist (UTF-8). */
  content: string
}
```

```ts type-equiv
/**
 * Save-time storage namespace for a spilled artifact. The session id lets a
 * backend group storage under the producing session, but the returned
 * {@link SpillLocator} is the model-facing handle. Forked sessions inherit
 * locators already present in the seeded log; those artifacts are not copied or
 * re-owned, and spills produced after the fork use the child session id.
 */
interface SpillOwner {
  sessionId: SessionId
}
```

保持期間に基づくクリーンアップは、古いロケーターを他の古いセッション成果物とともに期限切れにしうる。スピル seam はセッションごとのクリーンアップ・ポリシーを定義しない。

```ts type-equiv
/**
 * Producer of a spilled artifact. Tool results carry their model-issued call id;
 * session references identify the captured source session instead. Descriptive
 * source description only, never access control.
 */
type SpillSource = {
  kind: 'tool'
  /** The tool whose result was spilled (e.g. `web_fetch`). */
  toolName: string
  /** The model-issued call id the result belongs to. */
  callId: ToolCallId
  /** A short human label for the artifact (e.g. `result`). */
  label: string
} | {
  kind: 'session-reference'
  /** Session whose projected conversation was captured. */
  sessionId: SessionId
  /** Host-provided label for the referenced session. */
  label: string
}
```

## 結果

```ts type-equiv
/** A saved spill artifact: its locator, byte length, and backend-specific retrieval guidance. */
interface SpillRef {
  locator: SpillLocator
  bytes: number
  retrievalHint: string
}
```

`SpillLocator` はバックエンドが返す、[ブランド付き](core.md#branded-ids)のモデル向けハンドルである。ローカル・バックエンドはこれをファイルシステム・パスとして描画し、リモートやデータベースのバックエンドは URI、キー、コマンド・トークンとして描画できる。コンシューマーはこれを不透明なものとして扱い、`read` が常に正しい取得手段であると仮定せず、`retrievalHint` とともに描画する。

```ts type-equiv
/**
 * Opaque model-facing handle for one spilled artifact. A local backend may use a
 * filesystem path; a remote or database backend may use a URI or key. Consumers
 * render it with {@link SpillRef.retrievalHint}, but do not parse it.
 */
type SpillLocator = Branded<'SpillLocator'>
```

## サービス

`SpillStore`（`ctx.spillStore`、[`packages/spill/spill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill/src/index.ts) で定義）は 1 メソッドの抽象サービスである：`saveText(input) → Promise<SpillRef>`。完全な `content` を永続化し、実際のストレージ障害（権限、ENOSPC、バックエンド利用不可）では拒否する。seam はストレージだけを所有する。保持ポリシーも、ツール結果の置き換えも、取得／検索 API も持たない。

ローカル・バックエンド（[dsh-spill-local](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/spill/spill-local)）は `<root>/session-<hash>/<random>-<safeName>` の下に書き込む。設定済みまたは遅延作成されるプライベート（0700）のルート、`sha256(sessionId)` のセッション・サブディレクトリ、そして仕掛けられた symlink がリダイレクトできないよう、所有者のみの排他的（`open(path, 'wx', 0o600)`）書き込みである。その `locator` はローカル・パスであり、`retrievalHint` はそのパスに対して `read` または `grep` を使うようモデルに伝える。ポリシー・コンシューマー（[dsh-spill-policy](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/spill/spill-policy)）は、`maxInlineTokens` を超えるテキスト／画像の結果を、順序付きの先頭／末尾コンテンツとスピル・アドレスで置き換える。ベストエフォートであり、保存に失敗した場合は成功した呼び出しを `isError` に変えず、元のインライン結果を保つ。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxspillstore--spillstore-abstract-seam"></a>

### `ctx.spillStore` — `SpillStore`（抽象 seam）

抽象スピル・ストレージ・サービス。サブクラス化して saveText を実装し、そのサブクラスをプラグインとしてロードする。`ctx.spillStore` として登録される（コンテキストごとに 1 実装。2 つ目をロードすると cordis の標準的な重複サービス動作として例外を投げる）。

すべての実装が守らなければならないセマンティクス：

- saveText は完全な `content` をそのまま永続化し、不透明なロケーター、正確なバイト数、モデル向けの取得ガイダンスを返す。
- ストレージは要求の SaveTextSpill.owner セッションでスコープされる。バックエンドはプライベートな（誰でも読めるわけではない）場所と、呼び出し元の `suggestedName` から導出され、それと決して等しくならない衝突のない名前を選ぶ。
- `saveText` は実際のストレージ障害（権限、ENOSPC、バックエンド利用不可）で拒否する。呼び出し元がどのように劣化させるかを決める（スピル・ポリシーは拒否をベストエフォートとして扱い、インライン結果を保つ）。

```ts cordis-catalog
/**
 * Persist `input.content` to a session-scoped spill artifact.
 * @param input - the owner, caller-supplied source fields, suggested name, and full text to save.
 * @returns the saved artifact's {@link SpillRef}; rejects on a storage failure.
 */
abstract saveText(input: SaveTextSpill): Promise<SpillRef>
```

ソース: [`packages/spill/spill/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/spill/spill/src/index.ts)
<!-- END GENERATED cordis-surface -->
