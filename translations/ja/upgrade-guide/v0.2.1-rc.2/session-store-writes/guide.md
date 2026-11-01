---
kind: upgrade-guide
description: "store 所有の Session は、作成告知が完了する前と削除後の書き込みを拒否する。"
---
# store 所有の Session 書き込みは公開に従う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

store 所有インスタンスの `Session.append()` と `appendPluginRecord()` は、インスタンスが `SessionStore` に入り、同期の `session/created` dispatch が完了するまで書き込みを拒否し、離れた後も拒否する。store 所有権は `prepare()` または `enter()` で始まる。creation listener は append できない。告知が失敗すると書き込みは無効のままになる。独立した `Session.create()` と `Session.fromRestore()` インスタンスは、store observer に通知せずに書き込み可能なままである。constructor、restore、fork の履歴は通常どおりロードされる。Agent factory は setup を呼び出す前に `session/created` を完了し、`agent/created` は引き続き setup の後に続く。`session/created` の間、`ctx.agents.get(session.id)` は `undefined` を返すようになった。setup 内で登録された listener は、自身の Session の作成告知を見逃す。setup event は永続化された履歴だけでなく、ライブな Session subscriber にも届くようになった。

失敗した setup は constructor の履歴と、すでに永続化に書き込まれた setup event を保持する。したがって、fork を含む失敗した seeded creation は store された Session を残すことがある。その identity は新しい作成には利用できなくなる。

Permission のデフォルトは `agent/created` 中に初期化されるため、bare な Session を作成してもそれらは固定されなくなった。`dsh-permission-presets` は既存の依存関係に加えて `agents` サービスを必要とするようになった。

## 移行

1. 呼び出し元 fiber が所有する書き込み可能な Session には `ctx.sessions.create()` を使う。明示的なライフサイクル所有権が必要な場合は、append の前に `prepare()`、`enter()`、`announce()` を呼び、すべての終了 event が完了するまで返された disposer を保持する。
2. 独立した log 構築には `Session.create()` と `Session.fromRestore()` を残す。インスタンスを `enter()` に渡す前に履歴を構築するか、`prepare()` の seed として渡す。削除しても独立した書き込みアクセスは復元されない。コールドな永続書き込みは persistence の write handle を使う。
3. 書き込みと Agent 依存の作業を `session/created` listener から移す。書き込みには setup または `agent/created` を使い、Session 告知中に Agent を lookup する代わりに、setup では提供された Agent を、`agent/created` ではその payload を使う。
4. 自身の `session/created` を監視する setup インストールの observer を `agent/created` listener に置き換える。Session 作成 observer は、factory の作成が始まる前にインストールされた scope に保つ。
5. 履歴が永続化された失敗した作成の後は、同じ id で `create()` するのではなく `ctx.agents.resume({ resumeSessionId })` で再試行する。回復のために保存済み履歴を保持する。
6. `dsh-permission-presets` をマウントするカスタム profile は、`dsh-agent`（`agents` の provider）もマウントし、Agent を作成する前に permission plugin の activation を待たなければならない。
7. creation observer が期待する registry 状態を見ること、permission の初期化が完了すること、失敗した作成が保存済み履歴を resume できること、拒否された append が履歴を変えないことを確認する。
