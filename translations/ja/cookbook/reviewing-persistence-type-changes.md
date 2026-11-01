---
description: "プルリクエストを開く前に、Session 永続化型の変更をローカルで生成・確認・検証する。"
---

# クックブック: 永続化型の変更をレビューする

日本語 | [English](../../../docs/cookbook/reviewing-persistence-type-changes.md) | [中文](../../../docs/cookbook/reviewing-persistence-type-changes.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/reviewing-persistence-type-changes.md](../../../docs/cookbook/reviewing-persistence-type-changes.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

依存関係をインストール済みのコントリビューター・チェックアウトで、宣言された Session 永続化型を変更した後にこのチュートリアルを使います。バイリンガルの互換性説明を用意し、1つのコマンドに変更を分類させ、その記録を生成させます。[記録リファレンス](../persistence-changes/index.md)がファイルと自動ルールを説明します。比較入力はすべてチェックアウト内にあり、ベース・ブランチやネットワーク・アクセスは不要です。

## 目次

- [任意: 変更を検査する](#generate)
- [1. 変更を記録する](#acknowledge)
- [2. チェック、コミット、プッシュする](#verify)
- [未受け入れの記録を更新する](#competing-records)
- [開発メモ](#dev-note)

-----

<a id="generate"></a>
## 任意: 変更を検査する

記録前のプレビューには、リポジトリ・ルートから実行します:

```sh
pnpm --silent run verify-persistence-changes --json
```

JSON を消費するときは `--silent` を使います。そうしないと pnpm がライフサイクル失敗のテキストを stdout に追加します。失敗したコマンドは依然としてコード 1 で終了します。

報告された root、パス、変更種別、互換性レビューの要否を読みます。参照される型は複数のイベント・ダイジェストに影響しうるため、影響を受けるすべての root を検査します。履歴が新しいスキーマをカバーするまで検証は失敗します。古い生成インベントリーも検証を失敗させます。記録コマンドがそれを更新します。フィールドや union の選択肢を並べ替えた後に `changes` が空なら、`pnpm run gen-persistence-catalog` を実行してチェックを繰り返します。ダイジェストが変わらなければ、コピーされた宣言やソース位置がカタログ差分を生んでも、新しい確認は不要です。

PR をその確認履歴から独立してレビューするには、ベースとヘッドのインベントリーをローカル JSON ファイルとして保存し、実行します:

```sh
pnpm --silent run persistence-review --before .artifacts/base.schema.json --after docs/persistence-schema.json
```

記録された決定は、この比較から遷移を隠しません。`--check` は現在のソースと確認済み履歴だけを比較します。記録後は、base/head レビューを置き換えたり、人間が変更をレビューしたことを確立したりできません。

それらのファイルを供給したコミットをレポートとともに記録します。構造化出力には `--json` を追加します。この読み取り専用比較は、共有変更を影響を受ける root とグループ化し、union 位置ではなく実際のリテラル `kind`/`form` 値を使います。曖昧な選択肢は別々の追加と削除のままです。互換性セクションは各 root の権威ある分類結果をコピーします。構造の説明は確認チェックを置き換えません。現在のカタログ・ラベルと宣言名は説明的メタデータであり、構造アンカーとフィンガープリントが型を識別します。

すべてのインベントリーは[ノード・テーブル・レイアウト](../persistence-changes/index.md#files-and-ownership)を使い、両方の比較入力は現在のツールで生成されていなければなりません。`formatVersion` は正規化ルールを識別します。ストレージ・レイアウトは root フィンガープリントを変更せず、確認も要求しません。

<a id="acknowledge"></a>
## 1. 変更を記録する

最初に[受け入れ済みベースライン](../session-format-status.md#finalization-record)を確認し、そのロックされた記録を保持します。[互換性ルール](../persistence-changes/index.md#compatibility-rules)の下で両方のリーダー方向を評価します。構造差分は、安全な解釈または拒否の証拠があれば同一バージョンのままでいられます。自動的に高い writer バージョンを要求するものではありません。

`en` と `zh` を含むローカル JSON ファイルを書き、それぞれに `summary`、`compatibility`、`verification` の文字列を入れます。以下の入力は、実際に実行した required から optional へのフック監査フィールド変更を説明しています。説明とテスト証拠は自分の変更の事実に置き換えてください。CLI はこれらの主張を確立しません。

入力を `.artifacts/persistence-change.prose.json` として保存し、必要なら無視対象のディレクトリを作成します:

```json
{
  "en": {
    "summary": "Makes the persisted hook audit decision optional.",
    "compatibility": "Existing records remain valid. Hook execution consumes HookOutput instead of replaying this audit field. Producers still write decisions, and absence does not imply pass.",
    "verification": "pnpm exec vitest run packages/experimental/hook-protocol/tests/events.spec.ts: 10 tests passed."
  },
  "zh": {
    "summary": "将持久化的钩子审计决策改为可选。",
    "compatibility": "已有记录仍然有效。钩子执行消费 HookOutput，不回放此审计字段。写入方仍然记录决策，缺失不代表 pass。",
    "verification": "pnpm exec vitest run packages/experimental/hook-protocol/tests/events.spec.ts：10 个测试通过。"
  }
}
```

この例の id の代わりに、日付と説明的なスラッグを使います:

```sh
pnpm --silent run persistence-changes --record 2026-09-11-poc-optional --prose .artifacts/persistence-change.prose.json --json
```

コマンドは履歴とペアの散文を検証し、確立された同一バージョンの決定を推論し、書き込み前に明示的な互換性の決定をチェックします。記録ペアを生成し、スキーマ、両方のカタログ、マシン・インベントリー、ペアリング記録の後に完了します。コミット前に説明と返された `changes`、`roots`、`files` をレビューします。`--prose` を省略すると未完成のドラフトが作成され、説明が完成するまで検証に拒否されます。

`requiresCompatibilityReview` とマークされた変更では、確認の Compatibility セクションで両方のリーダー方向を説明し、Verification に実行済みチェックを記録します。その証拠がバージョンを維持することを支持する場合は `--decision same-version` を指定します。ヘッダー・バージョンが変わらず明示的な決定がないフラグ付き変更を記録または更新すると、`decision-required` で失敗します。構造差分だけではバンプを推論しません。実際のヘッダー・バージョン増加は `version-bump` を推論できます。実効的な識別子が安全でない解釈を防げない場合は、[Session 形式バージョンの追加](adding-a-session-format-version.md)に従い、`--decision version-bump` を選択します。その記録には独自の増加する `SessionHeader.version` 遷移を含めなければなりません。無関係な履歴バンプはそれを承認できません。通常の変更は別のベースラインを作成しません。

<a id="verify"></a>
## 2. チェック、コミット、プッシュする

[テスト方針](../testing.md)を通じて変更された所有者の振る舞いチェックを選択し、ドキュメント・チェックを実行します:

```sh
pnpm run doc-sync
```

`doc-sync` は永続化インベントリーとカタログの新しさ、完全な履歴、バイリンガル・ペアリングをチェックします。記録コマンドの `ok: true` は、これらのチェックや所有者の振る舞い・移行テストを置き換えません。JSON の失敗は `ok: false`、診断用の `code`、終了コード 1 を保持します。構造化された変更は安定した kind と root ごとの before/after ダイジェストを含むため、自動化は説明を解析する必要がありません。

記録生成は自身のカタログと記録ペアを所有します。パッケージ README や他のバイリンガル・ページへの編集は、引き続き通常のペアリング・ワークフローに従います。意図した差分をレビューしてステージし、通常どおりコミットとプッシュを行います。ステージ時の lint、ペアリング、空白フックと、プッシュ前の Host/Client 型チェックは引き続き適用されます。

<a id="competing-records"></a>
## 未受け入れの記録を更新する

記録後にソースが変わった場合は、互換性の説明をレビューし、同じ未受け入れの終端記録を更新します:

```sh
pnpm --silent run persistence-changes --update 2026-09-11-poc-optional --prose .artifacts/persistence-change.prose.json --json
```

コマンドはマシン宣言、スキーマ、カタログ、ペアリングを更新します。`--prose` なしでは既存の説明を保持します。その説明を完全な遷移に対して再評価し、フラグ付きの同一バージョン更新には `--decision same-version` を指定します。更新は初期ベースライン、別の記録が依存する記録、確定済みチェックポイント記録を拒否します。確定済みチェックポイント以外では、ツリーはレビュー受け入れを推論しません。受け入れ済み履歴を保持し、代わりに後継を作成します。

統合が競合する終端記録を作成した場合は、残りの履歴に対して未受け入れ記録を更新し、結果の差分を再評価します。無関係な root の確認を更新する必要はありません。[メカニズムの決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-09-11-persistence-type-history.md)が、完全なスナップショットと root ごとの先行世代を保持する理由を説明します。

レビューを必要とする既存プロパティ値型の変更では、まず散文ファイルの Compatibility と Verification セクションを修正し、実際の新しい遷移と実行済みチェックを説明します。以前の required から optional への説明を別の変更に再利用してはいけません。その後、その散文と明示的な同一バージョン選択で記録を更新します:

```sh
pnpm --silent run persistence-changes --update 2026-09-11-poc-optional --decision same-version --prose .artifacts/persistence-change.prose.json --json
```

<a id="dev-note"></a>
## 開発メモ

<details>
<summary>メンテナー向けの作業コンテキスト — クリックして展開</summary>

なし。

</details>
