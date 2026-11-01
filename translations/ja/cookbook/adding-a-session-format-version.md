# クックブック: Session ログ形式バージョンの追加

日本語 | [English](../../../docs/cookbook/adding-a-session-format-version.md) | [中文](../../../docs/cookbook/adding-a-session-format-version.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-session-format-version.md](../../../docs/cookbook/adding-a-session-format-version.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

リリース済みデータを書き換えずに次の Session ログ形式バージョンを導入するためのチュートリアルです。[バージョンとリリース状態の権威](../session-format-status.md)を読み、チェックアウトの writer と最新のリリース形式を特定します。N をその検証済みのリリース形式、N+1 をターゲットとします。名前とメタデータ内のこれらのプレースホルダーは数値に置き換えてください。動作するコントリビューターのチェックアウトから始め、[パッケージ・チェックリスト](adding-a-package.md)、[形式ライブラリ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format/README.md)、[リリース形式の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)を読んでください。

## 目次

- [1. バージョンとリリース基底を選ぶ](#choose-the-version)
- [2. 同一性エッジを追加する](#add-an-identity-edge)
- [3. アーティファクト単位のステージと検証を実装する](#stages-and-validation)
- [4. 現行バージョンのコンシューマーを更新する](#current-version-consumers)
- [5. スナップショットの後継を作成する](#snapshot-successors)
- [6. 統合結果を検証する](#validate)
- [開発者向け V4 コーパス試行](#v4-corpus-trial)
- [開発メモ](#dev-note)

<a id="choose-the-version"></a>
## 1. バージョンとリリース基底を選ぶ

形式をバンプするのは、実効的な識別子が安全でない解釈を防げない場合だけです。ヘッダー、イベント・エンベロープ、イベント・セマンティクス、再構築に対する構造的変更は、[リーダー互換性ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-10-08-session-reader-compatibility-review.md)の下での評価を必要とし、自動的なバンプは行いません。安全な同一バージョン内の進化は[型変更手順](reviewing-persistence-type-changes.md)で記録します。Session 形式の整数を、パッケージのリリース・バージョン、SQLite スキーマ・バージョン、projection ユニット・バージョン、プロトコル・ラッパー・バージョンと区別してください。

バンプが必要なとき、N を [version/status](../session-format-status.md) で受け入れ済みまたは公開済みのバージョンとします。V4 が受け入れられた後は、V4 の公開が記録される前でも、そのバンプは V5 を必要とします。レビュー済みの同一バージョン内進化にはこのステップは不要です。

N+1 には共有の `release/*` 統合基底を使います。基底の変更は writer、codec、カタログ配線、同一性移行、検証を追加します。各独立子ブランチはその基底から作成し、PR は別の独立子のブランチではなくリリース・ブランチをターゲットにします。各子は構造変換、バリデータ、コンシューマー、テストを同じ隣接移行パッケージに追加します。レビュー順序を表すためだけに追加のバージョンを割り当ててはいけません。レビュー済みの子を PR 経由でリリース・ブランチにマージし、リリース前に統合結果を検証します。リリース・ブランチの force-push 保護と削除保護を守り、force-sync してはいけません。

リリース済み codec は受け入れ済みの解釈を保持します。現行バージョンの codec は、履歴の可読性を保ちながら、明示的に判別されたバリアントを許容できます。古い codec を広げて現行機能を許容してはいけません。N→N+1 コンバーターは、N+1 出荷後に修正を受けたり、追加の履歴ケースをサポートしたりできますが、その出力が N+1 と互換でなければなりません。修正は以降の変換に影響し、既存の N+1 後継には影響しません。変更されたマッピングと既存出力の継続サポートを文書化し、新しいターゲット表現にはリーダー互換性ルールを適用します。

同一バージョンのログは隣接移行をバイパスし、resume 後に履歴と新しいレコードのバリアントを混在させることがあります。したがって各整数は、1つの固定スキーマではなく、サポートされる表現のファミリーを識別します。所有するリーダーはそれらのバリアントを保持して区別しなければならず、次の外向き移行がそれらを処理しなければなりません。同一バージョンの移行を追加したり、正規化を `restoreCurrent` だけに頼ったりしてはいけません。変換された検証を使う現行バージョン JSONL の読み取りは、そのフックをバイパスします。暫定の統合形式には、使い捨ての隔離された Harness home を使います。変更された隣接変換は、コミット済み世代を書き換えたり実際のユーザーの home を使ったりするのではなく、変更されていない履歴入力から再実行します。

writer を変更する前に、[アーカイブ・コマンド](../../../docs/persistence-changes/historical-formats/README.md#maintenance)を使ってその完全な永続化スキーマを保存し、`docs/persistence-changes/historical-formats/vN.*` 以下にバイリンガルの形式リファレンスを追加します。以前のレコードはすべて保持します。形式カバレッジ・チェックは、新しい writer 未満のすべての整数が独自のドキュメントを持つことを要求します。現在の生成カタログは新しい writer だけをカバーします。

<a id="add-an-identity-edge"></a>
## 2. 同一性エッジを追加する

パッケージ・チェックリストに従って、N→N+1 のライブラリを作成します。マウントされるプラグインではありません。同一性の本体変換は、初期の配線スキャフォールドにすぎません。[V2-to-V3 仕様](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format-v2-to-v3/README.md#v2-to-v3-specification)は、明示的な変換と保持ルールの固定された例であり、拡張したり同一性変換として扱ったりするエッジではありません。

`dsh.sessionFormatMigration` を、数値の `from: N` と `to: N+1`、export パス、export する migration、source codec、target codec、target-header バリデータ、target restorer とともに宣言します。前のエッジ・パッケージが export する source codec を再利用し、そのパッケージに依存します。リリース済み codec をコピーしたり再定義したりしてはいけません。target codec とバリデータは新しいパッケージから export します。エッジをカタログの直接依存として追加し、workspace の TypeScript paths とプロジェクト参照を追加します。

[core Session 型](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/types.ts)の `SESSION_FORMAT_VERSION` を、新しいエッジ宣言とともに N+1 に設定し、カタログを生成します。以下のコマンドは宣言されたチェーンだけを生成し、新しいバージョンを実装しません:

```sh
pnpm run gen-session-format-catalog
```

[ジェネレーター](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/gen-session-format-catalog.ts)は、ゼロから writer バージョンまでの各ステップにちょうど1つの隣接パッケージを要求し、ディレクトリ名／パッケージ名、隣接 codec の export、宣言された依存関係が一致していなければなりません。ギャップ、重複または余分なエッジ、未知のメタデータ・メンバー、Session を peer と開発依存の両方で共有しないカタログを拒否します。`generated.ts` を手で編集するのではなく、宣言を修正してください。カタログはビルド時に静的であり、プラグインのマウントが履歴の可読性を決定してはいけません。

<a id="stages-and-validation"></a>
## 3. アーティファクト単位のステージと検証を実装する

[Stage インターフェース](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format/src/types.ts)を使い、アーティファクト全体の配列から配列への移行器は使いません。不変の `SessionFormatMigration` 宣言が `migrateHeader`、`validateTargetHeader`、`createStage` を提供します。`createStage` の各呼び出しは、1つのソース・アーティファクト用の独立した状態を作成します。カウンター、保留中のイベント、参照マップはそこに保持し、可変ステージを Session 間で共有してはいけません。

`transformEvent(event, context)`、`transformRun(run, context)`、`finish(context)` を実装します。`context.emitEvent` または `context.emitRun` を通じて同期的に emit します。1回の呼び出しは0個、1個、または多数の出力を生成できます。ステージは codec 所有のコンパクトな run を直接消費するか、中間配列を実体化せずに `run.expand()` を反復できます。スケジューリングは呼び出し元が所有し、チェーンは下流ステージの前に上流ステージを完了します。

継承されたカットは物理行数ではなく論理イベント数として扱います。`headerInheritedEventCount` は EOF 前に判明している場合にのみ公開します。`finish` は正確なターゲット・カットを返します。先行するカーディナリティ変更エッジは、構築時にそのカウントを利用できなくすることがあります。必要な場合は検証済みシード・マーカーから導出し、直接の N 入力だけでなく、サポートされる各履歴世代から N+1 へのシード付きマルチホップ復元をテストします。未知のカットにゼロを代入してはいけません。

新しいエッジのイベント受け入れと変換ルールを明示的に定義します。[V2-to-V3 ソース監査](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format-v2-to-v3/README.md#source-audit)と[alpha V0→V1 ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-alpha-historical-unknown-event-refusal.md)は、新しいエッジではなく、それらのリリース済みエッジのポリシーを所有します。どちらもすべてのエッジに一般化してはいけません。構造やイベント位置の変更は、ソース・イベント、ペイロード・メンバー、参照を分類し、不透明データが有効なままでいられるかを明示的に決定することを要求します。[同一バージョン保持](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-30-retain-ignorable-external-session-events.md)だけでは、構造変換が安全であることを証明しません。ターゲットのセマンティクスを検証し、新たに受け入れる各ケースに拒否する反例を与えます。サポートされない変換を隠すために古いエッジを広げてはいけません。

`appendPluginRecord()` が書き込むプラグイン・レコードは、experimental パッケージが型ごとに読み戻す `plugin:` 名前空間の無視可能イベントです。ターゲット形式が表現できる場合はその名前のまま引き継ぎ、そうでなければどのレコードを落とすかをエッジ README に明記します。[無視可能イベントの決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-30-retain-ignorable-external-session-events.md)はそれらをベストエフォートで保持します。保存されたレコードが[experimental 永続化カタログ](../../../docs/experimental-persistence-catalog.md)に宣言を持たない場合でも、このポリシーを適用します。

ネイティブ writer の出力を変換のオラクルとして使います。同じ記録済み LLM メッセージ、ツール出力、その他の入力を古い writer とターゲット writer の両方でリプレイします。移行された古い出力を、形式正規化ヘルパーの前の生のターゲット出力と比較し、文書化された揮発性フィールドだけをマスクします。V4 では、通常のツール結果は通常のコンテンツ・ブロックを持つフラットな tool-role メッセージになります。理論上許される古い構造だけでは、新しいコア・コンテンツ型を正当化しません。

alpha コンバーターは、実装され証拠付けられたマッピングがない履歴ケースを明示的に拒否できます。それ以外では有効なログも含まれます。ソースは変更せず、部分的な後継も公開しません。サポートを拡張するときは実際のコーパスと再現可能な writer を優先し、ユーザー・コーパスは読み取り専用で調査し、サニタイズされた回帰フィクスチャを作成します。

エッジ README には、ターゲット形式が必要とする識別子とフィールドのマッピングだけを、それぞれの理由とともに文書化します。拡張可能であるという理由だけで、直接のソース種別や通常のメタデータに名前空間を追加してはいけません。不透明なネスト・データは保持します。alpha コンバーターは、解釈を発明するのではなく、新しいターゲット上の意味を得る予期しないソース・フィールドを拒否できます。

新しいエッジの README を、[V2-to-V3 仕様](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format-v2-to-v3/README.md#v2-to-v3-specification)に従い、その唯一の完全な変換・受け入れカタログとして維持します。変換されるすべてのヘッダー、イベント、メッセージ・スロット、ペイロード・フィールド、正確なリネームや衝突ルール、保持される id、座標、参照、継承カット、外部前提条件、拒否と不透明データのポリシー、サポートされないまたは延期された振る舞いを列挙します。履歴ソースの変換とネイティブ・ターゲットの受け入れを分け、各チェックをどの codec、restorer、インストール済みバリデータ、復旧ポリシーが行うかを明記します。宣言されたスキーマより意図的に狭い検証は記録します。このカタログはコードと同じ変更で更新し、生成されたスキーマ・インベントリーや PR の説明で置き換えてはいけません。

`sessionFormatCatalog.createRestore(header, { recovery: 'strict', validation: 'current' })` を通じた厳密な復元を証明します。行を順番に渡し、`finish()` を呼びます。これにより物理デコード、完全なチェーン、インストール済みの現行 Session 検証が実行されます。本番の recoverable/transformed ポリシーは、厳密なフィクスチャ検証と公開検証の代替にはなりません。エッジが実際に行うより厳しいソース検証を主張するのではなく、文書化された履歴検証の例外を保持します。

<a id="current-version-consumers"></a>
## 4. 現行バージョンのコンシューマーを更新する

Session の作成／復元、JSONL ファイル名の選択と公開、カタログの現行エンコーダー／復元器、projection キャッシュの世代同一性、リプレイとスナップショットの正規化、[物理フィクスチャ・レイアウト検証](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/session-fixture-layout.ts)、TypeScript/Python SDK の記録を含む、各現行バージョン・コンシューマーを追跡します。値が現行を意味する場合は writer 定数を使い、リリース済み codec と履歴フィクスチャにはリテラルの履歴バージョンを保ちます。現在のドキュメントと生成リファレンスは、その所有者を通じて更新します。

無関係なバージョンを自動でバンプしてはいけません。リクエスト・ラッパーの `sessionFormatVersion` は、埋め込まれた Session 世代を識別します。その外側のスキーマ・バージョンには独自の意味があります。projection ユニット状態のバージョンも同様に、キャッシュの Session 世代同一性を置き換えません。

読み取りと書き込みの両方のパスを検証します。ヘッダーのみの一覧は本文を読んだり公開したりしてはいけません。履歴の読み取りオープンは、書き込まずに移行済みのインメモリ・アーティファクトを返してかまいません。書き込みオープンは、append の前に最終的な現行後継だけを検証して公開しなければなりません。ソースのパス、バイト、inode は変更されません。より新しい、または無効な選択世代が、先行世代へのフォールバックを引き起こしてはいけません。公開タイミングは[準備の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-05-read-only-session-migration-preparation.md)が所有します。

<a id="snapshot-successors"></a>
## 5. スナップショットの後継を作成する

[スナップショットの所有権](https://github.com/deepseek-ai/deepseek-harness/blob/master/snapshots/AGENTS.md)と[スナップショット・ライブラリ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/test-support/session-snapshot/README.md)を読みます。[コーパス・ポリシー](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/session-snapshot-corpus-policy.ts)は宣言されたベースラインをリプレイ入力として保持するため、アップグレードは隣接チェーンを実行します。writer のバンプはそのコーパスの更新を要求しません。現行世代の後継が必要な場合は所有シナリオを選択し、各履歴ファイルを保持し、ターゲット・バージョンの正規の親ファイル名と子ファイル名を使います。先行世代をターゲット・ファイル名にリネームしたり、ヘッダーだけを変更したりしてはいけません。

変更されていないリプレイ入力には、所有者でキーなしリフレッシュを使い、その後 write-back なしでリプレイします。これらの SDK コマンドは `text-turn` とチェックアウトの writer バージョンを使います。N+1 を実装して配線してから、それらを使ってそのバージョンを生成します。機能については実際に影響を受ける所有者を選択します:

```sh
pnpm run test:snapshot:refresh snapshots/sdk/sdk.snapshot.ts -t text-turn
pnpm run test:snapshot snapshots/sdk/sdk.snapshot.ts -t text-turn
```

新しい世代、リクエスト・サイドカー、プロトコル出力をまとめてレビューします。すべての先行世代がバイト単位で同一のままであり、親子の役割が連続していることを検証します。選択は数値上最も高い世代を使うため、所有者の選択済み親への共有参照を更新します。packed-layout 移行器をバージョン・アップグレーダーとして使ってはいけません。モデルのトランスクリプトを変更する必要がある場合、シナリオ所有者は[テスト方針](../testing.md)の下でライブ記録を使い、必要なプロバイダー・キーを用意します。

保持ベースラインより古いケースは、`snapshot.yml` の `sessionFormat.version` とサポートされる `coverage` 名を通じて明示的に保ちます。記録とリフレッシュは固定された Session フィクスチャを変更しません。現行世代向けに[コーパス・ポリシー](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/session-snapshot-corpus-policy.ts)を更新しつつ、焦点を絞った direct-edge、multi-hop、packed-row、retry/failure、shipped-profile のカバレッジを維持します。コーパスと両方の SDK projection を確認し、検証失敗を黙らせるためだけに無関係なシナリオを大量リフレッシュしてはいけません。

<a id="validate"></a>
## 6. 統合結果を検証する

リポジトリ・ルートから実行します。これらのコマンドはカタログ宣言、Stage 構成、リリース済み V2→V3 エッジ、世代選択をチェックします。それらはベースラインであり、新しいエッジ用の焦点を絞ったカバレッジを追加してください:

```sh
pnpm run verify-session-format-catalog
pnpm exec vitest run scripts/gen-session-format-catalog.spec.ts packages/session/session-format/tests packages/session/session-format-v2-to-v3/tests packages/session/session-format-catalog/tests
pnpm run test:snapshot scripts/session-snapshot-corpus.corpus.ts
```

新しいエッジを実装した後、実際のテスト・パスを焦点を絞った Vitest 実行に追加します。実際の差分が選択する変更済み JSONL、リプレイ、projection、SDK テストを追加し、そのパスが変わる場合はビルド済み publication-Worker のスモークも追加します。厳密な移行の成功、スケルトンの同一性保持、不正および未知の必須イベントの拒否、決定論的な繰り返し復元、独立した並行ステージ状態、シード付きマルチホップ・カット、変更されていない先行世代、フォールバックなしを要求します。推測されたフルスイート結果ではなく、正確なコマンドと失敗を報告します。

冗長な決定記録を追加するのではなく、[所有する Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)を更新します。[リリース記録](../session-format-status.md#updating-the-record)は公開まで変更せず、公開後に検証済みのリリース証拠で更新します。関連するアクティブなノートを監査して置き換えを確認し、独立した根拠は保持し、アーカイブ済みノートは凍結したままにします。バイリンガルの散文を一緒に更新し、リポジトリ・ツールで各変更ペアを再記録してから、ドキュメント・チェックを実行します:

```sh
pnpm run verify-translation-pairing --write docs/cookbook/adding-a-session-format-version.md
pnpm run test:docs
pnpm run doc-sync
pnpm run lint
git diff --check
```

<a id="v4-corpus-trial"></a>
### 開発者向け V4 コーパス試行

writer が V4 の、依存関係をインストール済みのコントリビューター・チェックアウトから、一度限りの[移行スクリプト](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/migrate-sessions-to-v4.ts)を使います。開始前にターゲット・ルートを使う DSH プロセスを停止し、writer ロックと変化する子ログが移行を妨げないようにします。リポジトリ・ルートから実行します:

```sh
pnpm run migrate:sessions-to-v4
```

既定のルートは `~/.dsh/sessions` です。別のコーパスには `--sessions-dir /path/to/sessions-copy` を使い、使い方は `--help` で確認できます。並行ジョブ数は既定で利用可能な CPU 数（最大16）です。`--jobs N` は任意の正の安全な整数（より大きなエキスパート向け上書きを含む）を受け付け、`--jobs 1` は直列実行します。境界付きキューはコーパスのサイズに関係なく、その数だけ Session を同時に開きます。各履歴 Session は通常のロックされ検証された公開パスを通り、変更されていないソースファイルの隣に V4 後継を作成します。既存の V4 Session は読み取り専用で開かれ、再実行しても再変換されません。スクリプトはモデル・リクエストを行わず、変換や拒否のルールも変更しません。

ターミナルは各 Session の選択ファイル、バージョン、進捗、結果、経過時間を報告します。個々のエラーは後続の Session を停止しません。最終サマリーはすべての失敗と、システム一時ディレクトリ下の完全な診断ログを列挙します。失敗があれば非ゼロの終了ステータスを返します。また JSON を出力し、同じレポートを `migration.log` の隣に `summary.json` として保存します。ソース・バージョンと結果別のカウント、グループ化された失敗理由、入力ごとの診断、ランタイム詳細、両方のレポート・パスを含みます。既知の診断はイベント型、予期しないメンバー、子ログの関与、シーケンス番号を明示します。認識されないエラーは、推測された原因を付けずにメッセージを保持します。移行問題を報告するときはこれらのレポートを含めてください。この実行中に変換された Session は、すでに V4 にあるものとは区別されたままです。

並行変換では、子が V4 を公開すると親の子証拠が無効になることがあります。そのソース変更エラーだけが、初期ジョブがすべて終了した後に1回の逐次リトライを受けます。破損、サポートされない形式、保持されたロックは直ちに報告されます。完了行は元の入力インデックスと別個の完了カウントを保つため、順序が入れ替わった結果も読めます。

<a id="final-v3-vocabulary"></a>
### V4 公開前の最終 V3 イベント語彙

公開形式が V3 のままで統合 writer が V4 の間も、master は有効な V3 イベント型を追加できます。各 master 統合後、最初の V4 公開前に、移行が所有する `RELEASED_V3_EVENT_TYPES` を、そのリフレッシュが使った最新の V3 writer コミットと比較します。ローカルで検証済みの `origin/master` がまだ V3 を書いている間に、その完全な不変 id を取得します:

```sh
V3_SOURCE_REF="$(git rev-parse --verify 'origin/master^{commit}')"
pnpm run verify-v3-event-vocabulary --source-ref "$V3_SOURCE_REF"
```

検証ツールはそのローカル・コミットだけを読み、イベント名の完全一致を要求します。欠落した V3 名、余分にコピーされた名前、V3 以外のソース writer を拒否します。特に V4 専用の `developer/message` は凍結された V3 セットに入ってはいけません。各差分を V3 ペイロード、必要な変換、ターゲット受け入れ、コンシューマーに対してレビューします。名前を追加するだけでは完全な移行を確立しません。新しいソース・イベントを受け入れる前に、対応する移行回帰を追加します。

このコマンドは fetch もリモートの新鮮さの確立も行いません。リリース担当者は、選択したコミットが統合に含まれる最新の V3 writer であることを検証しなければなりません。古いリモート追跡 ref や、古いピンに対する合格結果では不十分です。確認した id をリリース検証とともに記録します。master がすでに V4 を書いている場合は、現在の master を解決するのではなく、記録済みの最終 V3 コミットを再利用します。V4 出荷後は、V3 語彙をその履歴ソースに結び付けたままにします。以降の V4 イベント追加はそれを更新しません。通常の静的 CI は可変の master ref に依存しません。

<a id="dev-note"></a>
## 開発メモ

なし。
