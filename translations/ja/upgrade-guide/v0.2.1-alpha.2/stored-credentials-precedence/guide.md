---
kind: upgrade-guide
description: "store がマウントされている場合、保存済み credential が環境変数のフォールバックより優先される。出荷される headless profile は保存済み認証を省略する。"
---

# 保存済みキーはエクスポートされた変数より優先される

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

以前のリリースでは、`$DSH_HOME/.credentials.yaml`（デフォルトでは `~/.dsh/.credentials.yaml`）の同じ名前の `refs` エントリより前に、エクスポートされた環境変数を使っていた。これは `DEEPSEEK_API_KEY=… dsh`、shell profile の export、CI secret、コンテナの `-e` 値に適用された。名前がエクスポートされている間、Models ページはそのキーを保存できなかった。

現在は保存済みエントリが優先される。エクスポートされた変数は、ファイルがその名前の値を保存していない間のみ適用され、`$cwd/.env` と `$DSH_HOME/.env` は引き続き両方の後にのみ適用される。Models ページは、名前がエクスポートされている間もキーを保存・置き換えできる。

出荷される `headless` profile は代わりに credential store、authorization、DeepSeek account provider を無効にする。起動環境、プロジェクトの `.env`、Harness-home の `.env` の順を使い、`.credentials.yaml` には触れない。保存済み API キーとサインインは、デフォルトでは headless job の認証に使われなくなった。Web、SDK、ACP は store を維持する。

harness home が `refs` の下に値を保存し、`dsh` を起動するプロセスが同じ名前をエクスポートしている場合に影響を受ける。そのような home をマウントするコンテナも含まれる。リクエストはエクスポートされたキーではなく保存済みキーを使うようになる。`.credentials.yaml` のない新しい harness home で起動する CI job やコンテナは影響を受けない。[保存済み credential の優先順位](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-10-09-stored-credential-precedence.md) の Agent Note が理由を記録している。

## 移行

1. `$DSH_HOME/.credentials.yaml` の `refs` 下の名前と、shell、CI job、コンテナがエクスポートする変数を比較する。片方にしか現れない名前は対応不要である。
2. 両方に現れる各名前について、それが望む account であれば保存済みキーを残す。そうでなければ、`refs` 下のその名前の行を削除する。実行中の harness はファイルをリロードするため、次のリクエストは再起動なしでエクスポートされた変数を読む。
3. headless job では、必要なキーをエクスポートして `dsh headless "<task>"` を実行する。保存済みキーやサインインを使い続けるには、3行の [headless 復元パッチ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/README.md#credentials) を適用する。storage を復元すると saved-first の優先順位も復元される。既存の profile、home、コマンド overlay は出荷済みの行を override できるため、有効な構成を確認する際は `dsh headless --dump-config` を調べる。
4. 意図したリクエスト account と、必要な箇所で起動プロセスが空でない値をエクスポートしていることを確認する。credential 値を出力せずに名前と存在を確認する。保存済み credential のみに依存する headless 実行は失敗するようになった。storage を復元するか、サポートされる環境認証を提供すると解決する。
