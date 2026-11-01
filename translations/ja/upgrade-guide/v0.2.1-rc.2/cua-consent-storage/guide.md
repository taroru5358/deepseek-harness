---
kind: upgrade-guide
description: "Computer Use のアプリケーション同意は、profile 設定から永続的な profile スコープの storage へ移る。"
---

# Computer Use の同意ストレージ

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Computer Use は `storageDomain` を通じてアプリケーションの同意を保存する。デフォルトの JSON バックエンドでは、各 canonical profile ディレクトリが独自の `$DSH_HOME/storages/cua_permissions_<sha256>.json` を持つ。「Allow for this session」は、DSH の再起動と Computer Use の無効化をまたいで conversation ID に従うようになった。別の conversation を許可するものではない。検証できないアプリケーションには引き続き一致するネイティブプロセス identity が必要で、そのアプリケーションを再起動すると新しい同意が必要になる場合がある。

`allowedApps` はレガシーな import source である。最初の store アクセス時に一度だけ import する。以降の設定編集はアクセスを許可・取り消しせず、取り消された grant は古い設定を再ロードしても復元されない。permission ページは grant と refusal の作成インターフェースのままである。macOS の権限は変わらない。

## 移行

1. profile の既存の storage サービスをマウントしたままにする。出荷される profile はすでに `storageDomain` と JSON バックエンドを含む。カスタム profile も同じサービスを必要とする。domain storage のない profile は authorization 操作を拒否する。
2. Computer Use を有効にし、その詳細を開いて storage を初期化する。常に許可されたアプリケーションが表示されることを確認する。
3. 確認後、profile 設定とその他の overlay の CUA エントリから `allowedApps` を削除する。残りの CUA 設定は保持する。
4. permission ページを使ってアプリケーションを取り消すか、再度尋ねることを許可する。古い実行中プロセスが保持する conversation 専用 grant は回復できない。アップグレード後に一度付与する。新しく付与された conversation アクセスは再起動を生き延びる。
5. storage ファイルをその profile の状態とともに保持する。profile を別の実ディレクトリに移すと、別の store が選択される。file lock は1台のマシン上のプロセスを調整し、ネットワークファイルシステムを共有する異なるマシン間では調整しない。
