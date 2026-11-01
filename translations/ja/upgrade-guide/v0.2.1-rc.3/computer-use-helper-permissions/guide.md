---
kind: upgrade-guide
description: "Computer Use は、DSH を起動するターミナルではなく、自身の helper アプリケーションの権限を必要とする。"
---

# Computer Use helper を認可する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

macOS の Computer Use は LaunchServices 経由で起動される `DSH Computer Use.app` として実行される。ターミナルや Desktop launcher に対する Accessibility と Screen Recording の grant は、この helper へのアクセスを許可しない。ソースからの準備ではアプリケーションをビルドして署名する。インストール済みパッケージは準備済みアプリケーションとその `current.json` receipt を必要とする。bare なネイティブバイナリを指すカスタム `helperPath` は、サポートされるネイティブ起動パスではなくなった。

## 移行

1. Computer Use はオンデマンドでインストールされるようになった。**Plugins → Add plugin** で `@deepseek-ai/dsh-cua-bundle` をインストールして有効にする。以前に出荷版コピーを選択していた profile はこのインストールが必要である。要求されたら DSH を再起動する。Computer Use plugin ページで、macOS が要求したときに `DSH Computer Use.app` に Accessibility と Screen Recording へのアクセスを付与する。
2. 自動準備を使うには、古い `helperPath` override または `DSH_CUA_HELPER` 値を削除する。カスタムビルドでは、完全なアプリケーションバンドル内の `DSH Computer Use.app/Contents/MacOS/dsh-cua-native` の実行ファイルを指す。
3. ネイティブアプリケーションの task を再開する前に、permission ページが `DSH Computer Use.app` を識別し、両方の権限が許可されていることを確認する。target アプリケーションの同意は引き続き必要である。
4. ターミナルや Desktop launcher に Computer Use のためだけにこれらの権限が付与されていた場合は、System Settings でそれらの古い grant を取り消す。helper を認可しても他のアプリケーションの既存権限は取り消されない。モデルが実行するプログラムは、引き続きその launcher のシステム権限と DSH sandbox policy に従う。

ソースビルドは ad-hoc 署名を使い、コード変更後に再度認可が必要になる場合がある。リリースのパッケージャーは安定した Developer ID 署名 identity を保持し、配布物を notarize しなければならない。[ネイティブビルドの手順](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua-macos/native/README.md#build-and-service) を参照。
