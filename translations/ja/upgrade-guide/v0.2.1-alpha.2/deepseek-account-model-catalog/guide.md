---
kind: upgrade-guide
description: "DeepSeek account route は models カタログを受け付けなくなり、account listing がモデルを提供し、modelOverrides がそれらを調整する。"
---

# DeepSeek account のモデルは account listing から取得する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`deepseek-account` route は以前、`llm-deepseek-account` エントリの `models` フィールドからモデルカタログを読んでいた。Models ページは account に対してこれを編集できた。route は現在、account の `GET /models` endpoint からモデルを列挙し、listing がロードされるまで組み込みカタログを提供し、モデルごとの調整は `modelOverrides` を通じてのみ受け付ける。Models ページはこの route のエディタを提供しなくなった。列挙されたモデルがより新しい DeepSeek Harness を必要とする場合がある。そのようなモデルは非表示になり、そのモデルへのリクエストは `CLIENT_UPGRADE_REQUIRED` で失敗する。エントリに `models` フィールドが残っている場合（以前の Models ページエディタが保存したものを含む）、plugin はロード時に `llm-deepseek-account: models is no longer configurable` で停止し、Desktop と Web はそのフィールドが削除されるまで account モデルを表示しない。サインイン済みの account で保存済みカタログがない場合は対応不要である。

## 移行

1. `$DSH_HOME/profiles/<profile>/cordis.patch.yml` を開き、`id: llm-deepseek-account` のエントリを見つける。
2. その `models` フィールドを削除する。フィールドを変更した各モデルについて、変更したフィールドのみを `modelOverrides.<model id>` の下に移す:

   ```yaml
   - id: llm-deepseek-account
     config:
       modelOverrides:
         deepseek-flash:
           maxTokens: 131072
   ```

3. DeepSeek Harness を再起動し、DeepSeek account にサインインして、モデルセレクターが account モデルを列挙し、Models ページが account をサインイン済みとして表示することを確認する。列挙されたモデルが拒否する override、または不明なモデルを指定する override は、Models ページに account の診断として表示される。
