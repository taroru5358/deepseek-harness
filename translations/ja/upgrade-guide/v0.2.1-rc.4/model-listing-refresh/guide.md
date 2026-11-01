---
kind: upgrade-guide
description: "DeepSeek のモデル一覧は初回使用後に自動更新され、カスタムの listing-cache owner はライブ設定と変更通知を提供しなければならない。"
---
# モデル一覧は自動的に更新される

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

DeepSeek account と API-key route は、初回使用後にモデル一覧を自動的に更新する。アイドル中も含まれる。`modelsMaxAgeMs` は10分のデフォルトを維持し、別のアクセスを待つ代わりにバックグラウンドリクエストをスケジュールするようになった。失敗した更新は、badge を含む最後に成功した一覧を保持する。成功した更新は badge メタデータを置き換え（省略や `null` も含む）、開いている Web selector に通知する。

`ModelListingCache` の consumer は `settings` と `changed` callback を提供しなければならない。その `dispose()` は、active な load と自動 credential lookup が完了した後に settlement する promise を返すようになった。`ListedCatalog` も `badges` map を必要とし、`ModelSelectInjected` は `resolveText` callback を必要とする。

## 移行

1. 既存の `modelsMaxAgeMs` 値を保持し、自動更新に同じ間隔を使う。各アクセスで更新し、自動 timer を実行しない場合にのみ `0` を使う。
2. カスタム `ModelListingCache` owner は、最新の route 設定のための `settings: () => ListingSettings | undefined` と、catalog consumer に通知する `changed: () => void` を提供しなければならない。route が listing に従わなくなったら `settings` から `undefined` を返す。
3. 所有 plugin の cleanup から `cache.dispose()` を返すか await する。
4. `ListedCatalog` を構築するコードは、badge がない場合は `badges: new Map()` を、または model id から `LlmModelBadge` への map を含めなければならない。デコードされた listing エントリからこの結果を構築するには `resolveListedCatalog` を優先する。
5. `ModelSelectInjected` を構築するコードは、所有 locale サービスから `resolveText: text => ctx.locale.resolveText(text)` を提供しなければならない。テストダブルは、意図的に英語を表示する場合に `text => typeof text === 'string' ? text : text.en` を使える。
6. 次の更新後、すでに開いているモデルセレクターがページを再ロードせずに現在の badge を表示することを確認する。badge フィールドを削除するか `null` に設定してクリアする。
