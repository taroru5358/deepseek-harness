---
kind: upgrade-guide
description: "SDK run はネイティブコマンドを認識し、入力の完了後に RunResult を返す。"
---

# SDK run はネイティブコマンドを受け付ける

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Python と TypeScript の `run()` は、既存のエントリを通じて文字列と content-block 配列を受け付け、常に `RunResult` を返す。サポートされるネイティブコマンドは、その定義 identity と有効な登録名によって認識される。英語と中国語の alias は名前変更された定義に従う。無関係な完全一致名の置き換えは通常の入力のままである。[サポートされるコマンドの規則](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sdk/server/README.md#command-input) を参照。

`final_response` / `finalResponse` は最終的な assistant テキストを含み、activity が何も出力しない場合は空文字列を含む。state-only コマンドはネイティブ handler の結果テキストを返す。`finish_reason` / `finishReason` は終端結果を報告する。すべての結果は Session id、root event、Session-tree 通知を保持する。state-only コマンドはモデル turn を待たずに終了する。`/plan <task>` は、ネイティブ review と承認された継続を含む、admission された activity を待つ。goal の作成と resume は、goal が停止し、その closing activity が終わるまで、自動ラウンドをまたいで待つ。低レベルの strict command 呼び出しは、ネイティブ handler の settlement 後に引き続き `CommandExecution` を返す。

content-block 配列はネイティブルーティングをバイパスしなくなった。認識されたコマンドは1つの text block と任意のエンコードされた raster image を必要とし、ネイティブ handler が画像 admission を強制する。画像付きの `/plan off` は plan 状態を変えずに失敗する。空配列と空白のみの入力は Session 作成前に拒否され、画像のみの入力は引き続き受け入れられる。strict endpoint による identity 拒否は、handler を実行したり prompt を送信したりせずに、初期化済みの Session を保持する。

## 移行

1. prompt とコマンドに対して既存の `result.final_response` / `result.finalResponse` アクセスを維持する。失敗については `finish_reason` / `finishReason` を調べる。コマンド id とネイティブ結果は `command/run` と `command/done` event に残る。継続や復元のために結果の Session id を保存する。
2. plan review のために user-question callback を生かしておく。状態変更コマンドの失敗は `RunResult` に現れる。ルーティング、不正なプロトコルデータ、transport の失敗は引き続き raise または reject する。timeout や observer の失敗は runtime の作業をロールバックしない。
3. リテラルのキュー入力には Python の `client.session_prompt(session_id, blocks)` または TypeScript の `client.prompt(sessionId, blocks)` を使い、その後の activity を管理する。runtime の skill 認識は引き続き適用される。
4. カスタム `session/query` client は `{ sessionId, contentBlocks }` を送る。コマンド応答はネイティブ handler を acknowledge し、対応する `session.query.completed` 通知が最終的な入力結果を含む。通知の順序を保持し、送信前に subscribe する。
5. `run()` を呼ぶ前に空の入力をフィルタする。画像には画像対応モデルと既存の `run([text_block, image_block])` 形式を使う。Python の不正なネイティブコマンド応答は、validation の cause を保持したまま `SdkProtocolError` を raise する。呼び出し元が選択する汎用 response model は、その validation 動作を維持する。
