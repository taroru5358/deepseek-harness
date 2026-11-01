---
kind: upgrade-guide
description: "HTTP fetch provider は非公開の宛先を拒否しなくなり、HttpFetchProvider はアドレスリゾルバーを受け付けなくなる。"
---
# `web_fetch` はプライベートな宛先に到達する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-web-fetch-http` は以前、各ホスト名を解決し、loopback、private、link-local、reserved のいずれかの応答を `WEB_BLOCKED_URL` で拒否し、接続を検証済みアドレスに固定していた。現在はすべてのリクエストを、インストールされた proxy policy とシステムリゾルバーを通じて送るため、`web_fetch` は loopback、プライベートネットワーク、クラウドメタデータのホストに到達する。fake-IP DNS を使う TUN モード proxy の背後でも、設定なしでフェッチが機能する。

このパッケージは `HttpFetchResolver` をエクスポートしなくなり、`HttpFetchProvider` コンストラクターは `HttpFetchLimits` のみを取る。

provider がモデルをプライベートサービスから遠ざけることに依存していたデプロイと、リゾルバーを `HttpFetchProvider` に渡していたコードが影響を受ける。

## 移行

1. モデルがプライベートな宛先に到達してはならない場合は、プロセスのネットワークアクセスを制限するか、それらの `web_fetch` URL を拒否する `tools/pre-execute` policy を追加するか、`@deepseek-ai/dsh-tool-web` の fetch tool を構成から削除する。
2. `new HttpFetchProvider(limits, resolveAddresses)` を `new HttpFetchProvider(limits)` に置き換える。テストリクエストをローカルサーバーに向けるには、解決を override する代わりに URL にサーバーのアドレスを入れる。
3. `pnpm run typecheck` で移行を確認する。`HttpFetchResolver` への参照が残っていないこと。
