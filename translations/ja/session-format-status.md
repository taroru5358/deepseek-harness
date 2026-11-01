# Session 形式のバージョンとリリース・ステータス

日本語 | [English](../../docs/session-format-status.md) | [中文](../../docs/session-format-status.zh.md)

> これは非公式の日本語訳です。英語版（[docs/session-format-status.md](../../docs/session-format-status.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

このリファレンスは、チェックアウトのライター、受け入れ済みの互換性ベースライン、最新の公開 Session 形式を区別するために使います。ライターはコード定数が所有し、下の確定記録とリリース記録が、受け入れ済みの履歴と公開のエビデンスを別々に示します。他のドキュメントはこれらの値を再掲せず、ここへリンクします。

## 目次

- [信頼できる情報源](#sources-of-truth)
- [確定記録](#finalization-record)
- [リリース記録](#release-record)
- [記録の更新](#updating-the-record)
- [開発メモ](#dev-note)

<a id="sources-of-truth"></a>
## 信頼できる情報源

- **チェックアウトのライター:** [core Session types](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/src/types.ts)の `SESSION_FORMAT_VERSION` は、コード内で手作業で維持される唯一の現在ライターの番号です。[カタログ生成器](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/gen-session-format-catalog.ts)が codec の順序を導出し、隣接する移行がこの値に到達することを検査します。パッケージのバージョン、codec のエクスポート名、フィクスチャのファイル名、プロジェクション・キャッシュのバージョンは、ライターの権威ではありません。
- **最新のリリース形式:** 次の記録の `latestReleasedVersion` が、公開された Session 形式を示します。`evidenceTag` は、タグ付けされたライターがその値を持つ公開済み製品リリースを名指しします。その形式を初めて搭載したリリースである必要はありません。バイリンガル版は同じ記録に対して検査され、別の決定として維持されることはありません。
- **リリース・ステータス:** ライターの定数を、検証済みのリリース記録と比較します。等しければ、ライターの形式は出荷済みです。ライターのバージョンがより大きい場合は、まだ公開済みとして記録されていません。その確定記録は、受け入れ済みの互換性ベースラインを独立に示します。古いチェックアウトを、より新しいブランチの検証済み記録と比較する場合、ライターのバージョンが小さいことは古いライター形式を示します。ローカルの整合性ゲートは、1 つのチェックアウト内でその順序を拒否します。別の released boolean は維持しません。より大きいバージョンを未リリースと宣言する前に、公開済みリリースが記録を進めていないことを確認してください。

alpha、beta、リリース候補の製品公開は、リリース済み Session 形式の義務を確立します。GitHub の prerelease フラグは、永続化されたユーザー・データを使い捨てにはしません。リリース記録の欠落は、未公開のエビデンスではありません。[バージョニングと権威の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md)が互換性の決定を担い、[リリース済み形式の移行](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)が不変の世代と隣接する変換を担います。

[形式リファレンス](../../docs/persistence-changes/historical-formats/README.md)は、ゼロからチェックアウトのライターまでのすべての整数を、履歴スキーマと既存の現在カタログとともに記述します。

<a id="finalization-record"></a>
## 確定記録

```yaml session-format-finalization
latestFinalizedVersion: 4
```

V4 は[チェックポイント](../../docs/persistence-changes/finalized/v4.json)に受け入れ済みの互換性ベースラインを持ちます。[リーダー互換性のルール](persistence-changes/index.md#compatibility-rules)を満たし、必要なレビューが記録されていれば、スキーマ変更は V4 のままでかまいません。バージョンを上げる決定には、より高いライター・バージョンとそれ自身のヘッダー遷移が必要であり、受け入れ済みの 3→4 遷移を再利用することはできません。受け入れ済みの機械記録と after スキーマは不変のままです。

確定は、将来の V4 のすべての追加を凍結するものではなく、公開を主張するものでもありません。下のリリース記録は、独立に検証された公開済みバージョンを保持します。受け入れ済みの意味を保つ通常のコメント、エイリアス、ソース位置、実装の修正は、このベースラインを変えません。

[最終 V3 の語彙](cookbook/adding-a-session-format-version.md#final-v3-vocabulary)は履歴として残り、現在の V4 の追加とは独立です。

<a id="release-record"></a>
## リリース記録

```yaml session-format-release
latestReleasedVersion: 4
evidenceTag: dsh-v0.2.0-rc.2
```

エビデンス: 公開済みの製品タグ `dsh-v0.2.0-rc.2`。タグ付けされたライター: `packages/core/session/src/types.ts`。

<a id="updating-the-record"></a>
## 記録の更新

Session 形式のバージョンアップが実装されたら、コード定数と隣接するカタログを一緒に更新します。公開前にこのリリース記録を進めてはいけません。製品リリースがより高い Session 形式を初めて公開したら、公開とタグ付けされたライターを確認し、同じバイリンガル更新でこの記録とエビデンス・タグ、タグ付けされたライターのパスを進めます。同じ形式を搭載した後続の製品リリースでは、記録を変える必要はありません。開発トランクで記録を下げてはいけません。

[ドキュメント標準のテスト](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/doc-standard.spec.ts)は、記録の構造、バイリンガルの等価性、エビデンス・タグとライター・パスの整合性、記録されたリリースがチェックアウトのライターを超えないことを検査します。このキーなしの検査は GitHub に問い合わせたり、記録が最新であることを証明したりしません。公開の検証はリリース更新の一部として残ります。

一般的な挙動には「現在の形式」と「次に隣接するバージョン」を使います。固定された移行の入力と出力、ワイヤー・スキーマ、履歴のエビデンス、それら特定バージョンのテストには、明示的な番号を保ちます。[形式バージョンのクックブック](cookbook/adding-a-session-format-version.md)は、最新の確定済みまたはリリース済み形式に N、その後継に N+1 を使います。

<a id="dev-note"></a>
## 開発メモ

なし。
