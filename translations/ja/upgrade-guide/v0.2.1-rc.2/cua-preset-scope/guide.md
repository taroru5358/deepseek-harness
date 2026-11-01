---
kind: upgrade-guide
description: "standalone のネイティブ CUA 構成は、グローバル tool 登録を明示的に選択しなければならない。"
---

# CUA は PTC session scope をデフォルトにする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

ネイティブ CUA provider は `agentPreset` を `ptc` にデフォルトする。既存と新規の PTC Agent に tool を登録し、保存済みの application-grants 設定が新しいフィールドを省略している場合も同様である。Agent preset registry のないカスタム構成は、デフォルトでグローバル CUA tool を受け取らなくなった。

## 移行

1. 標準の Computer Use スイッチではデフォルトを維持する。既存の application grant は引き続き使用できる。
2. グローバル tool を必要とする standalone 構成では、profile パッチまたは overlay のネイティブ CUA エントリの `config` の下に `agentPreset: false` を追加する。`allowedApps` や `helperPath` を含む他のフィールドは保持する。
3. 再起動するか HMR に編集を適用させ、意図した Agent が native モードまたは PTC SDK を通じて `cua` を公開することを確認する。
