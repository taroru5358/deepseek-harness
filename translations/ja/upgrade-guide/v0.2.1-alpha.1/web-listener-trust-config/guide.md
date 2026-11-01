---
kind: upgrade-guide
description: "Web profile は `webRuntime` サービスと `web-runtime` 行の `trustedHosts` 設定を `webStartup` に置き換え、ワイルドカードの listener host を拒否する。"
---

# Web listener と信頼設定は `webStartup` へ移る

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Web profile は `webRuntime` サービスを提供しなくなり、`web-runtime` 行は `trustedHosts` 設定値を宣言しなくなった。`webRuntime` を inject する profile、overlay、`--patch` ファイルは、決してマウントされないサービスを待つようになり、`ctx.webRuntime.trustedHosts` を読む式は評価に失敗するため、必須の Connection が起動できない。

`web-startup` 行は、呼び出しの `--trusted-host` authority を保持する単一の `webStartup` サービスを提供し、`connection` 行は `ctx.webStartup.trustedHosts` からそれらを読む。listener host はローカルインターフェースの具体的な IPv4 または IPv6 アドレス1つでなければならない。`host: 0.0.0.0`、`--host ::`、その他すべての unspecified address はロード時に拒否される。すべてのインターフェースに bind すると、ポートがネットワークに公開されるためである。

## 移行

1. 呼び出しの信頼を必要とするすべての行で `inject: [webRuntime]` を `inject: [webStartup]` に置き換える。
2. `ctx.webRuntime.trustedHosts` 式を `ctx.webStartup.trustedHosts` に置き換え、存在しなくなった `ctx.webRuntime.lanAddresses` を削除する。
3. `connection` overlay を `webStartup` 用に書き換え、`web-runtime` 行が持っていた `trustedHosts` 値をその式に移す。パッチは一致した行の `config` 全体を置き換えるため、その config を完全に再記述し、配列値の `!!js` 式は引用されたスカラーとして書く:

   ```yaml
   # before
   - id: connection
     inject: [webRuntime]
     config:
       trustedHosts: !!js "['app.internal', ...ctx.webRuntime.trustedHosts]"
   # after
   - id: connection
     inject: [webStartup]
     config:
       trustedHosts: !!js "['app.internal', ...ctx.webStartup.trustedHosts]"
   ```

4. ワイルドカード `host` をローカルインターフェースの具体的なアドレス1つに置き換える。bind IP 自体は `--trusted-host` エントリなしで Host fence に受け入れられるが、proxy や DNS authority には引き続き1つ必要である。
5. 移行後の overlay で起動する: `dsh --profile web --patch ./extra.yml --no-open` が `dsh web:` URL 行を出力して配信しなければならない。`webRuntime` を待ち続ける行は必須の Connection を pending のままにし、activation 失敗を報告する。拒否された `host` はロード時に失敗する。`dsh --profile web --patch ./extra.yml --dump-config` は起動前に構成済みパッチを出力する。[Web bundle README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md) が構成済みの行を所有する。
