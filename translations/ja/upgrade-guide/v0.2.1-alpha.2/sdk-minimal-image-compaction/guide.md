---
kind: upgrade-guide
description: "sdk-minimal のデフォルト tool 構成に read_image が加わり、古い context を自動的に要約する。"
---

# SDK-minimal の画像読み取りと自動 compaction

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`sdk-minimal` は persistent shell と `working_directory` に加えて `read_image` を広告する。context 圧力が設定されたしきい値に達すると、自動 compaction は古い履歴を要約する追加のモデルリクエストを送る。完全な Session log は非圧縮 JSONL として保存され続ける。route の予算を超える画像は attachment/path placeholder に恒久的に置き換えられ、リクエストを再試行できる。SDK の prompt テキストは `/compact` を実行しない。

tool 名を固定したり、prompt サイズをベンチマークしたり、モデルリクエストを数えたりする client は、これらのデフォルトを考慮しなければならない。画像読み取りには画像入力を宣言する adapter route が必要である。設定とモデルへの影響については [bundle リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) を参照。

## 移行

1. 新しいデフォルトを受け入れるか、以前のデフォルト tool を維持して自動 compaction を無効にするには、次のパッチを `$DSH_HOME/profiles/sdk-minimal/cordis.patch.yml` に置く:

```yaml
- id: tool-fs
  config:
    read: false
    readImage: false
    write: false
    edit: false
- id: compaction-basic
  config:
    auto: false
```

2. 新しい SDK runtime を起動する。追加のユーザー plugin がなければ、そのモデルリクエストには persistent な `bash` または `pwsh` tool と `working_directory` が含まれ、自動要約リクエストは送信されない。
