---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-16-session-format-v4

日本語 | [English](../../../docs/persistence-changes/2026-09-16-session-format-v4.md) | [中文](../../../docs/persistence-changes/2026-09-16-session-format-v4.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-16-session-format-v4.md](../../../docs/persistence-changes/2026-09-16-session-format-v4.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

確定した V4 ライターのために、宣言された SessionHeader.version を 3 から 4 へ進め、第一級の tool ロール結果とプロデューサー所有のソースを記録し、turn/end.reason に forked バリアントを追加する。履歴上のリクエスト・ヘッダーに束縛された名前のみのツール追加、ツール削除、遅延ローディングのスキーマ・マーカーを伴う developer ロールの Session 変更を追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-16-session-format-v4
baseline: false
changes:
  - root: "SessionHeader"
    previous: "2026-09-11-initial"
    after: "1a3440e3577382704d42a6263aa463504eb74c566734a55e9503a63efcd02445"
    decision: version-bump
  - root: "event:agent/inbox/spliced"
    previous: "2026-09-14-image-offload"
    after: "1506a9b8224986c83015ae99d2cb5ede705538c58c063d6a48ef6d761a31ba6c"
    decision: version-bump
  - root: "event:assistant/attempt"
    previous: "2026-09-14-image-offload"
    after: "15d5dfdd822aa35e115afd74a8982825a493880457774e6850bc1520b50875e4"
    decision: version-bump
  - root: "event:assistant/message"
    previous: "2026-09-14-image-offload"
    after: "1033093edd0db80ff410e00830b523405e00bb0c7684948e531ff65095799625"
    decision: version-bump
  - root: "event:compaction/summary"
    previous: "2026-09-14-image-offload"
    after: "e2f9a41e0989f54ed8cee80f8db2bcf9d60a5c810dc9d45b83fa050b9dce7602"
    decision: version-bump
  - root: "event:developer/message"
    previous: null
    after: "eef4ef54dc7a133d47448a4ee822e45a351314923ef5f66db34c8b24e4b32d80"
    decision: version-bump
  - root: "event:request/header"
    previous: "2026-09-11-initial"
    after: "4208123b50df5006b181481ab45fcf1cde807b88d3fd4d340090bc2e202fac41"
    decision: version-bump
  - root: "event:session/title-llm-request"
    previous: "2026-09-14-image-offload"
    after: "fa8f7d3ebf08a76c7f7a8b0781873c4d819b964da5dbb52cd3cdfa5da34f452d"
    decision: version-bump
  - root: "event:system/message"
    previous: "2026-09-14-image-offload"
    after: "69081694be231d56fd9580ba14645fd5e35373202605d5c5c841a9435b5fa3b1"
    decision: version-bump
  - root: "event:team/message/queued"
    previous: "2026-09-14-image-offload"
    after: "21fb6a90d5068f6a0003b7ab316ed2f56342477146a65c00db0f13c4d8df667d"
    decision: version-bump
  - root: "event:tool/ptc-dispatch"
    previous: "2026-09-14-image-offload"
    after: "100f6dca1468538239522cde3533e5bd721d0f1a7b50bea8b0eb533ea6c96163"
    decision: version-bump
  - root: "event:tool/result"
    previous: "2026-09-14-image-offload"
    after: "7c9f44e90a0058f4cc532ae20dad0c10afa6eba22e70a6c79fc79490bad64397"
    decision: version-bump
  - root: "event:turn/end"
    previous: "2026-09-14-image-offload"
    after: "0f8512903d94f57a4748fa1a2092e64342856796684e6b8343db685b192745ce"
    decision: version-bump
  - root: "event:user/message"
    previous: "2026-09-14-image-offload"
    after: "3f72db3d87a0c5c43e68be467b4cca728eaf5adc1d5d2b6975ff42bfbd961761"
    decision: version-bump
```

<a id="compatibility"></a>
## 互換性

tool ロールの宣言は 10 個のイベント・ルートを変更する。inbox エントリ、メッセージ・イベント、compaction サマリー、タイトル・リクエスト、チーム・メッセージ、PTC ディスパッチが、共有の `Message` または `ContentBlock` 宣言を埋め込んでいるためである。その union から `tool-result` を削除し、メッセージのロールを特殊化すると、10 個の独立したイベント・プロトコルを追加することなく、それらの到達可能なスキーマが変わる。`turn/end` の変更は forked の理由を別途記録し、`SessionHeader` はバージョンの引き上げを記録する。

[ネイティブ V4 検証の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-17-native-v4-read-validation.md)が、これらの現在のフィールドが必要とするリーダー受け入れを担う。

V3 から V4 への移行は、リリース済みのユーザー・ロールのツール結果を、必須の toolCallId とオプショナルの isError を持つ tool ロールのメッセージへ持ち上げる。ツール結果のラッパーはコンテンツ・ブロックの union から外れる。移行は、受け入れられたすべてのソース・イベントと継承されたカットを保持し、同じ永続化ルート内に保持された直接の子ログから、欠けている親の subagent/catalog レコードを追加する。履歴の本体の復元には明示的な子エビデンスの集合が必要であり、バックフィルできる子がない場合は空の集合を含む。欠けている、複数ある、またはバージョン不明の子記述子は、その子のバックフィルをスキップする。同一性、タイムスタンプ、モードが競合する場合は、公開せずに移行を拒否する。既存のカタログの事実は残る。履歴の読み取りオープンはメモリ内で結果を準備する。書き込みオープンは、変更されていない前身ファイルの隣に現在の後継を公開する前に、子の所属とリビジョンを再検証する。配送世代のチェックにより、履歴の承認がアクティブな V4 ウォーターマークになるのを防ぐ。V3 のリーダーは新しい世代を拒否する。確定した遷移は turn/end.reason に forked を追加し、正確なカットの fork は継承されたマーカーの後に子所有のエラー結果とクローザーを追加する。V4 は、決定的なブランチ固有の ID と文言を持つ、検査済みの not-started fork 結果を受理する。リリース済みの V0–V3 バリデータと記録済みの前身世代は変更されない。

`request/header` スキーマはまた、廃止された `system` キーを禁止として記録する。この宣言は、保存データやプロンプトの再構築を変えることなく、既存のネイティブ・リーダーの拒否を捉える。後で値を許可するには、通常のオプショナル・フィールド追加として分類するのではなく、同一バージョンの変更としての互換性レビュー、または明示的なバージョン引き上げが必要である。

プロデューサー所有のソースは、[V3 から V4 への移行](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-format-v3-to-v4/README.md#v3-to-v4-specification)を通じてリリース済みのプラグイン・ラッパーを置き換える。凍結されたリネーム表と衝突ルールは、ソースのフィールドとイベント座標を保持する。未知のプロデューサー帰属は、自身のすべての JSON プロパティを保持する。ネイティブの読み取りと書き込みのオープンは、Session を公開する前にソース・フィールドを検証する。既存の V4 ファイルは、受信側の移行エッジを再実行しない。

core 所有のユーザー・ソース・プロパティは帰属保持ポリシーを記録する。tmux-context はプロデューサー・ローカルの重複抑制を保ちつつ、その位置の帰属を修飾する。Auto Review と compaction サマライザーはリクエスト専用のユーザー入力を使い、履歴移行のサポートを削除せずに、それらのアクティブなソース登録を削除する。これらの入力は永続的な Session メッセージとして書き込めない。カタログの formatVersion 2 はポリシーのメタデータを保存するが、Session バージョンを変えたり、凍結されたスキーマ記録を書き換えたりはしない。system、model、tool のソースは、それぞれの厳格な意味ルールを保つ。

developer イベントは元のロールを保ち、オープンな step を要求する。各追加は toolName を保存し、developer/message.headerSeq は、完全に一致する ToolSchema をちょうど 1 つ含む、以前に既知の request/header を識別する。1 つのイベント内のすべての追加はその履歴ヘッダー・リビジョンを共有し、削除のみのメッセージやその他の developer メッセージは headerSeq を省略する。ネイティブと Session の受け入れは、未知の ignorable レコードを解釈したり、無関係な JSON メタデータを落としたりすることなく、欠落、前方参照、非ヘッダー、未知ヘッダー、曖昧、定義不完全、廃止されたインライン定義の形式を拒否する。同名の置換、再起動、fork、サーフェス置換、compaction は、最新のヘッダーやレジストリを参照せずに履歴スキーマの同一性を保つ。汎用の sourceEventSeqs は独立したままである。developer ソースは、ユーザー・ソースと同じ記録済みの帰属保持ポリシーを使う。オプショナルの deferLoading マーカーは追加履歴と独立している。出荷されるプロファイルは developer レコードを出力しない。プロバイダーのシリアライズ、自動出力、UI のサポートは引き続き延期されている。

PTC プロデューサーは `source.kind: 'ptc-mode'` を書き込む。V3 から V4 へのエッジは、`tools-code-mode` と `tools-ptc` を履歴のプラグイン検索キーとして保持し、両方をその現在の種別にマップする。ソース・ポリシーは `ptc-mode` を帰属のみの修飾に対して予約する。

<a id="verification"></a>
## 検証

正確なカットの fork を統合した後、対象を絞った Session、agent-loop、Session Controller、V4、chat-view、compaction のスイートが 63 ファイルにわたる 1,523 件のテストに合格した。V4 の fork テストは、エンコード、デコード、復元を通じて元の ID とテキストを保ち、不正な形式の結果を拒否し、ネストした継承カットを検証する。tool ロール移行のテストと SDK スナップショットの更新も元の変更で合格した。ビルド済みの Python ランタイムの sdk-snapshot シナリオが合格し、対象を絞った pi-ai と auto-review のカバレッジが、影響を受ける 3 つのモジュールを 100% カバーして 351 件のテストに合格した。

生成された request-header 予約のリグレッションと、既存の廃止構文および Session サーフェスのテストが、3 ファイルにわたる 65 件のテストに合格した。生成されたフィールドはオプショナルの `never` を保つ。現在の分類器は、オプショナルの文字列を許可することを互換性レビュー必須とマークするが、ネイティブ・リーダーは引き続き廃止されたキーを拒否する。

プロデューサー・ソースとリクエスト入力のチェックは、12 ファイルにわたる 447 件のテストに合格し、プロバイダーの等価性、リクエスト専用型の拒否、ユーザー帰属の保持、ソース移行、ネイティブ・ソースの受け入れを含む。生成されたインベントリーを tool ロールの親と比較すると、4 個の変更されたルートと 447 個の変更されていない型フィンガープリントが報告される。

対象を絞った Session、V4、リクエスト入力のスイートは 38 ファイルにわたる 884 件のテストに合格し、V3 から V4 へのパッケージ全体と Session サーフェスについて、ステートメント、ブランチ、関数、行の 100% をカバーする。カバレッジには、履歴の同名スキーマ束縛、複合的な追加と削除、不正な形式の参照と定義、未知ヘッダーの拒否、不透明な未知レコード、メタデータの保持、fork、置換、compaction 参照、および世代を書き換えないネイティブの plain/zstd 読み書き受け入れが含まれる。

<a id="dev-note"></a>
## 開発メモ

なし。
