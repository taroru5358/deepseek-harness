---
kind: upgrade-guide
description: macOS Computer Use の grant は bundle identifier を使い、古いアプリケーション identity には同意の更新を必要とする。
---

# macOS Computer Use のアプリケーション identity

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

アプリケーションの grant は、大文字小文字を区別するネイティブ bundle identifier を使う。異なるパスにあるバージョンやコピーは、署名にかかわらず権限と取り消しを共有する。bundle identifier のないアプリケーションは canonical な実行ファイルパスを使い、always-allow grant も受け取れる。そのパスで実行ファイルを再起動または置き換えても同意は保持され、移動すると同意の更新が必要になる。有効な実行中 bundle identifier は実行ファイルパスを必要としない。Grant レコードは executablePath を省略でき、その場合 permission ページはパスの tooltip やアイコンなしで名前を表示する。既存レコードは引き続き読めるが、このフィールドを必要とする古いバージョンは新しい pathless レコードを読めない。profile の storage の場所と conversation scope は変わらない。

これは、パス/署名 grant を保存している macOS ユーザー（レガシーな `allowedApps` 設定から import された grant を含む）に影響する。それらの grant は新しい identity に一致しない。表示と削除は可能なままであるが、他のインストールを自動的に認可しない。古いレコードと新しいレコードが同じ名前とパスを表示することがあり、permission ページはそれらを区別しない。承認済みの bundle identifier を宣言する別のプログラムや、承認済みの fallback path で実行ファイルを置き換えるプログラムは権限を継承する。このポリシーは publisher を認証しない。

## 移行

1. Computer Use のアプリケーション管理で、影響を受ける各アプリケーションの既存の always-allowed エントリをすべて取り消してから、再度選択する。アップグレード後にすでにアクセスを付与している場合は、古いレコードと新しいレコードを区別しようとせず、そのアプリケーションのすべてのエントリを削除する。
2. 「Applications not allowed」の下で、そのアプリケーションに「Allow asking again」を使い、以前の拒否や取り消しが次の prompt を妨げないようにする。アプリケーションを再度選択し、意図した scope を承認する。「Always allow」は、profile 内で同じ bundle identifier を持つすべてのバージョンとコピーを対象とする。
3. その bundle identifier を持つ別のインストール済みコピーを選択し、付与された scope 内で再度尋ねられないことを確認する。共有 grant を取り消すと、すべてのコピーへのアクセスが停止する。異なる bundle identifier を持つバージョンは引き続き個別の承認を必要とする。
