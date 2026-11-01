# AGENTS.md — ドキュメント標準

日本語 | [English](../../docs/AGENTS.md)

> これは非公式の日本語訳です。英語版（[docs/AGENTS.md](../../docs/AGENTS.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このファイルは、ドキュメントの構造、Markdown の階層、執筆ルール、`verify-doc-budgets` の上限を定義します。配置と検証には [dsh-doc](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-doc/SKILL.md) を、必要なカバレッジと編集上の判断には [dsh-prose-standard](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-prose-standard/SKILL.md) を使います。[アーカイブ済みの doc-tiers の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/process/2026-07-04-doc-tiers-and-budgets.md)が、歴史的な根拠を記録しています。

## ドキュメントの構造

これらのルールは人向けのドキュメントに適用されます。[Agent Notes](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/README.md) はその対象外のままです。[postmortem](postmortem/index.md) はインシデントに限定したリファレンスであり、時系列は教える順序ではなくエビデンスを記録します。ドキュメントの主題とツリー上の位置がそのスコープを定めます。自身の主題を適切な詳細で記述し、直接の子については目的、責任、高レベルの挙動だけを述べます。より低レベルの詳細は、所有する子孫へリンクします。ドキュメントの種類がこのスコープを広げることはありません。リファレンスが網羅的でよいのは、自身の主題についてだけです。テストの仕組み、フィクスチャ、harness は最も低い所有階層に属し、上位のドキュメントはそこへリンクします。

対象範囲内のすべてのドキュメントを、チュートリアルかリファレンスに分類します。チュートリアルは成果への順序付けられた道筋に従い、各ステップが必要とするものだけを導入します。リファレンスは参照スコープと現在の挙動を定め、教える順序を持ちません。実質的なチュートリアルとリファレンスの内容は分け、どちらかの部分が小さいときはセクションにその旨を記します。

チュートリアルを書く前に、読者の出発点の知識と各概念を、初級、中級、上級のいずれかに非公開で分類します。依存する概念より先に前提を確立し、難易度を徐々に上げ、不要な上級の内容は後のチュートリアルやリファレンスへ移します。

この順序で執筆します。ツリー内でドキュメントの位置を定める。許される詳細度を決める。チュートリアルかリファレンスを選ぶ。チュートリアルなら、概念を前提と難易度で並べる。子孫が所有する詳細を移動する。下位レベルの説明を、その所有者へのリンクに置き換える。

## 階層の分類体系: 1 つの事実に 1 つの居場所

各事実には 1 つの居場所、すなわちその仕事を担う階層があります。それ以外の場所では、そこへリンクします。

| 階層 | 役割 | ここに置かないもの |
|---|---|---|
| ルート `AGENTS.md` | 常設の指示: agent がすべてのセッションでコンテキスト内に必要とするルール。各 1〜3 行で、その居場所へリンクする | 物語、実例、状況依存の手順、リンク先の居場所から再掲したもの |
| サブツリーの `AGENTS.md`（`packages/`、`docs/`、`.agents/notes/`） | そのサブツリー固有の指示 | ルートのファイルがすでに担うリポジトリ全体のルール |
| [architecture.md](architecture.md) | 順序付けられた地図: コンポジション、コア・パッケージ、ループ、seam、拡張ポイント。`packages/` を変更する前に読む | 型定義（→ サブシステム）、パッケージごとの詳細（→ パッケージ README）、決定の根拠（→ Agent Notes）、実装ステータスの注記 |
| [subsystems/](subsystems/index.md) | サブシステムごとに 1 ページのリファレンス: 型定義、セマンティクス、生成された Cordis API | 挙動の語り（→ architecture.md） |
| [Agent Notes](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/README.md) | 現行の決定記録: なぜか、何を諦めたか、必要な検証。`implemented/` のノートは出荷済みの現実を現在形で記述する | 移行計画、受け入れタスクのチェックリスト、フィクスチャのウォークスルー、決定が出荷された後の仕様口調（「〜すべき」）。アーカイブ済みノートは凍結された履歴であり、現在の権威ではない |
| [defensive-patterns/](defensive-patterns/index.md) | このリポジトリがレビューで強制するルール。成果物ごとに分割し、一般的なバグ分類の 1 ページとドメインごとのページを持つ | 型定義とセマンティクス（→ サブシステム）、決定の根拠（→ Agent Notes） |
| [postmortem/](postmortem/index.md) | インシデントの物語。物語形式の語りが属する唯一の階層 | — |
| [永続化の履歴](persistence-changes/index.md)と[形式リファレンス](../../docs/persistence-changes/historical-formats/README.md) | 型変更の承認、リリース間の比較、完全な履歴形式のスキーマ | 挙動だけの変更、現在のランタイム契約 |
| [cookbook/](cookbook/adding-a-package.md) | 番号付きの検証ステップを持つ、手順ごとのハウツー | 設計の根拠（→ 各ガイドがリンクする Agent Note） |
| [user/](user/index.md) | ドキュメント・サイトが公開する、製品向けのガイド | 生成されたリファレンス表、コントリビューター向けの手順、決定の履歴 |
| パッケージ README | パッケージごとの契約: 設定、セマンティクス、制限、拡張ポイント、[Model Experience](cookbook/adding-a-package.md#4-write-the-package-readme) | JSDoc の再掲、生成カタログの再掲（イベント／ツールの表）、他のパッケージの関心事 |
| [development.md](development.md) | コントリビューターのセットアップ、日々のワークフロー、CI の概要。[i18n 契約](../../docs/i18n/README.md)の下のバイリンガル・ペア | ランタイム／バージョンの根拠（→ Agent Notes）、`package.json` のスクリプトからずれるチェックごとの一覧 |
| 生成リファレンス: [subsystems/](subsystems/index.md) のページごとの `cordis-surface` 領域、[Cordis コア API と継承階層](../../docs/cordis-api/context.md)、[tool-catalog](../../docs/tool-catalog.md)、[config-catalog](../../docs/config-catalog.md)、[persistence-catalog](../../docs/persistence-catalog.md)、[module-graph.md](../../docs/module-graph.md)、[cli-help](cli-help.md) | ソースから再生成され、鮮度でゲートされる網羅的な英語ソース。レビュー済みの中国語版は[ペアリングのワークフロー](../../docs/i18n/README.md#scope-and-exclusions)に従う | 生成された英語ソースや領域への手編集。中国語版はペアリングを通じてのみ更新する |
| スキル（`.agents/skills/`） | 再利用可能なワークフローと専門的な判断基準 | 製品とランタイムの契約（→ ドキュメントまたはソース） |

配置: バグ → postmortem、根拠 → Agent Notes、手順 → クックブック、型定義 → サブシステム、パッケージの契約 → README、常設の指示 → ルート `AGENTS.md` と根拠へのリンク、レビューで強制するルール → `defensive-patterns/`。

## 執筆ルール

- **現在の状態を記述する。** 履歴はコミット、PR、Agent Notes、postmortem、スコープ付きの永続化記録に保ちます。その他の散文は、変更やスタック上の位置ではなく、生きた仕組みを名指しします。Session 形式に関する一般的な散文は[バージョン／ステータスの権威](session-format-status.md)へリンクし、バージョン固有の契約、例、エビデンスには番号を残します。
- **Agent Note の作成基準を適用する。** 機械的／ローカルな編集は対象外で、ローカルの UI 変更も含みます。既存の所有ノートは正確なまま保ちます（[スコープ](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/README.md#when-to-write-one)）。
- **段落ごとに物理的な 1 行**（`verify-md-wrap`）: エディターのソフト・ラップを使います。コード・ブロック、表、リストの構造は書式を保ち、コードのコメントはリンターの桁数制限の下に保ちます。
- **フェンス付きの `ts` ブロックはコンパイルできなければならない**（`doc-typecheck`）。貼り付けた型宣言と元の JSDoc には ` ```ts type-equiv ` を、本体を除いた公開クラス宣言には ` ```ts public-api ` を使い、どちらも manifest に登録してドリフトを防ぎます（[仕組み](development.md#documenting-types-verbatim-ts-type-equiv)）。
- **所有する[サブシステムのページ](subsystems/index.md)は、ドキュメント化された型の形を変える同じ変更で更新する。** `verify-type-equiv` が捉えるのはドリフトした貼り付けであり、一度もドキュメント化されていない新しい型ではありません。型は、それを宣言するパッケージ・グループのページでドキュメント化します（[ページのスコープ](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-subsystem-pages.ts)）。
- **ペアは一緒に更新する**: [用語ガイド](../../docs/i18n/terminology.md)に従った、単一パスのアクティブ agent の作業が、初出の注記を位置変更し、触れていない散文を保ち、記録し直します。`dsh-translate-docs` はユーザーが起動するままです（[契約](../../docs/i18n/README.md)）。
- **コメントと JSDoc は推論の記録ではなく、完全な契約を述べる。** 挙動、失敗、タイミング、所有関係、モダリティ、例外、帰結、自明でない方向づけを保ちます。語り、テストのウォークスルー、レビューの分析、コードの言い換えは削除します。ローカルの契約を保ち、その根拠へリンクします。詳細は [dsh-prose-standard](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-prose-standard/SKILL.md) を使います。
- 直接書きます。行為者と事実を名指しします（[リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-prose-standard/SKILL.md)）。`seam` は定義されたケイパビリティにのみ使います。比喩的な「ゲート」「語彙」「サーフェス」ではなく、正確なチェック、型、API、操作、挙動を名指しします。

## 語数の予算

[scripts/doc-budgets.manifest.json](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/doc-budgets.manifest.json) が常設ドキュメントの上限を定め、`pnpm run verify-doc-budgets` が超過や欠落したファイルを拒否します。

ゲートが赤になったとき:

1. 別の階層に属する内容を**移動**する。必要なら 1 行のリンクを残す。
2. ここに属するが短くできる内容を**圧縮**する。
3. 語数にその余地が必要なときだけ上限を**上げ**、PR で manifest の差分を正当化する。低すぎる上限は予算のバグです。

上限はガードレールであり、削減の目標ではありません。目標以下なら少なくとも 5% の余裕を保ち、目標を超えているなら、移動か圧縮でドキュメントが目標を下回るまで上限を凍結します。ドキュメントにまだ余裕があるときだけ上限を下げます。目標: ルート `AGENTS.md` は 2,080 以下、`architecture.md` は 2,400 以下、サブツリーの `AGENTS.md` は 600 以下（ただし `packages/AGENTS.md` は 810 以下、このファイルは 1,430 以下）、`packages/README.md` は 994 以下。加えて `cordis-primer.md` は 600、`testing.md` は 1,300、`examples/AGENTS.md` は 310。予算のない階層はレビューが統治します。

## slop チェックリスト

どのドキュメントでもこれらを探します。[dsh-doc](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-doc/SKILL.md) はこの一覧を監査として実行します:

- 重複したルール: 特徴的な語句を検索し、居場所を 1 つに保ち、残りはリンクする。
- 許されない階層にある履歴: 現在の事実を述べ、歴史的な所有者へリンクする。
- 散文や図の中の実装ステータスの注記（「実装済み!」「将来: …」）。ステータスは腐敗し、リポジトリのレイアウトとパッケージ manifest がそれを担う。
- ソースや生成器が権威であるのに、手で再掲したカタログ、JSDoc、テスト、パッケージ、ステータスの目録。
- 推論の記録: 実装のステップごとの語り、自明な分岐の証明、テストのウォークスルー、却下したローカルの代替案。結果としての契約や永続する根拠を保ち、導出に使った道筋は削除する。
- 兄弟メソッドのそばで繰り返される根拠。所有するケイパビリティやヘルパーで一度だけ述べる。
- 段落の壁: 複数のルールと括弧書きの余談を 1 つの段落が担っている。分割するか、詳細をその居場所へ降ろす。
- 強調のインフレ: 太字、大文字、どこでも「重要」では、何も目立たない。挙動を変える節にだけ強調を残す。
- `implemented/` の Agent Notes での仕様口調: 「〜すべき」、移行計画、受け入れチェックリスト。実装済みの Agent Note は、[implemented ノートの指示](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/AGENTS.md)に従って、現在あるものを記述する。

## リポジトリ参照

現在のファイルには相対 Markdown リンクを、履歴の参照にはタグや PR 番号を使います。`verify-md-links` はローカルのターゲットを検査します。[参照の検証](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-repository-references.ts)は、16 進数の文字を含むコミット識別子と、許可されない組織の URL を、維持されるファイル内で拒否します。数値データとの衝突を避けるため、10 進数のみのトークンは、たとえコミットを識別していても無視されます。
