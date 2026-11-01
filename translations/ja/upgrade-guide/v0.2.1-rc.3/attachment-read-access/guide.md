---
kind: upgrade-guide
description: "画像読み取りリクエストは完全な attachment reference を必要とし、Session membership を確認しなくなる。"
---

# 画像読み取りリクエストは完全な attachment reference を含む

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

0.2.1-rc.3 では、`session/attachment` は `{ sessionId, attachmentId }` を受け付け、完全な画像 reference を求めて要求された Session の履歴を検索していた。現在は `{ sessionId, attachment: ref }` を必要とする。`ref` は `attachmentId`、`mediaType`、`bytes`、`width`、`height` を含む `ImageAttachmentRef` である。Client の `session.readAttachment()` も ID の代わりに完全な reference を取る。

Host はこの reference を既存の `AttachmentStore.readImage(ref)` に渡す。保存されたオブジェクトに対して ID、実際の形式、バイト長、幅、高さを検証する。応答は引き続き `{ attachment, data }` で、`data` に base64 バイトを含む。提供された reference を保持し、任意の `name` と `originalDimensions` も含む。これら2つのフィールドは呼び出し元が提供する表示情報であり、ファイルに対して検証されない。欠落、破損、不一致のオブジェクトは明示的に失敗する。

Connection の認証と Gateway の権限チェックは引き続き適用される。提供された `sessionId` に対して endpoint を呼ぶことを許可された identity は、同じ store 内の画像を読み取れる。別の Session だけが参照している画像や、どの Session も参照していない画像も含まれる。handler は画像の membership や Session の存在を確認せず、認証後に Session が存在しない場合の `session/not-found` を返さなくなった。

## 移行

1. wire 呼び出しを `{ sessionId, attachmentId: ref.attachmentId }` から `{ sessionId, attachment: ref }` に更新し、message image block の完全な reference を使う。Client 呼び出しを `session.readAttachment(ref.attachmentId)` から `session.readAttachment(ref)` に更新する。
2. 返された `name` と `originalDimensions` を呼び出し元の表示情報として扱う。保存されたオブジェクトに対して検証されるのは ID、media type、バイト長、幅、高さのみである。
3. デプロイが画像を分離するために Session membership に依存している場合は、アップグレード前に別々の `DSH_HOME` ディレクトリを持つ別々の Host インスタンスを実行する。出荷される構成は、1つの Host 内で Session ごとに別々の store を選択しない。Gateway の権限は endpoint 呼び出しを制御し、画像の Session membership は制御しない。
4. 完全な reference を持つ許可されたリクエストが、提供された Session に存在しない画像を読み取れることを確認する。その Session が存在しない場合も含む。拒否された Gateway リクエスト、存在しない画像、本質的メタデータが一致しない reference は失敗しなければならない。既存の Session log と保存済み画像に移行は不要である。
