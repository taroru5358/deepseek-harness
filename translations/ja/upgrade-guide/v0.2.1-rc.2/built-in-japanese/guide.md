---
kind: upgrade-guide
description: "組み込みの言語登録は、同じ id を使うサードパーティの言語パックと競合する。"
---

# 組み込み言語は重複する言語パックを置き換える

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

locale plugin は `ja`、`es`、`fr`、`de`、`ru`、`pt` を登録し、`common`、`settings.locale`、組み込み UI plugin の辞書を持つ。これらの id を使うサードパーティの言語パックは、already-registered language または dictionary エラーで失敗するようになった。以前の日本語またはフランス語 README の例に基づくパックも含まれる。

## 移行

1. 言語パックの client エントリで、これらの id の `ctx.locale.addLanguage(...)` と、組み込み plugin が提供する namespace の辞書登録を削除する。自身の plugin が所有する namespace の翻訳は残す。
2. パックが異なる言語 variant を表す場合は、定義とその辞書の両方に、組み込み id ではなく `fr-CA` のような別個の BCP 47 形式 id を登録する。
3. パックを有効にしてアプリケーションを再ロードする。重複登録エラーなしで起動し、意図した言語を選択するとその翻訳が表示されることを確認する。
