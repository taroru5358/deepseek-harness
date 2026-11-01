---
kind: upgrade-guide
description: "オプションの hook、webhook、tool、skill に対する明示的なインストール、peer 依存関係、設定。"
---

# オプション機能の明示的なインストールと構成

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

CLI と Python runtime はインストール依存関係から hook bridge を省略し、CLI は webhook ペアを省略する。base と完全な Web preset は dormant な Ralph 行を省略し、base は dormant な badge 行を省略する。削除された行を有効にするだけのパッチは、その plugin を挿入しない。デフォルトの active 動作、plugin 登録名、tool 名、記録済み Session event 名は変わらない。

パッケージの所有権と命名は [experimental package policy](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/experimental-package-policy.ts) に従う。インストールと構成は別である。plugin パッケージをインストールしてもマウントされず、bundle を選択してからでないと、それが提供する行を設定できない。

## 移行

1. target profile に不足しているパッケージを、必要な peer とともにインストールする。profile は peer の自動インストールを無効にする。たとえば `dsh plugin --profile web add @deepseek-ai/dsh-hooks-claude-code @deepseek-ai/dsh-hook-protocol` を実行する。Codex bridge は同じ protocol peer を必要とし、GitHub adapter は `@deepseek-ai/dsh-webhook-github` とともに `@deepseek-ai/dsh-webhook` を必要とする。
2. 行を override する前に、利用可能な bundle を選択する。手動で構成する機能では、削除された行の override を完全な挿入に置き換える。preset ローカルの tool は、必要なサービスを含めてその preset の `config.plugins` 内に追加する。Host badge の挿入は次のとおり:

   ```yaml
   - insert:
       - id: skill-badge
         name: '@deepseek-ai/dsh-skill-badge'
   ```

3. GitHub review では、削除された CLI の例ではなく adapter とともに配布される例を使う。[インストール済み profile の手順](../../../user/guide/github-review.md) に従う。カスタマイズした rule module は、その import がインストール済みパッケージを解決できるよう profile 内に保つ。
4. profile を起動し、設定済みの plugin が missing-package、missing-peer、missing-row エラーなしで activate することを確認する。想定した Agent preset で期待する tool または skill が利用可能であることを確認する。
