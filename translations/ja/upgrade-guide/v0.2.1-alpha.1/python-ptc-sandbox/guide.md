---
kind: upgrade-guide
description: "Python PTC 構成は sandbox サービスを必要とし、直接の Python ファイル操作は解決済みの file policy に従う。"
---

# Python PTC は共有 file sandbox を適用する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-experimental-ptc-runtime-python` は `sandbox` と `sandboxPolicy` を必要とする。以前は、直接の Python ファイル操作が封じ込めなしで実行され、provider は明示的な `sandboxPolicy` 入力を拒否していた。現在は各プログラムに共有 policy を適用し、`PtcRunResult.sandbox` で封じ込めを報告する。制限付き実行は、封じ込めを確立できない場合に `sandbox-unavailable` で失敗する。

これは Python runtime を選択するカスタム構成に影響する。出荷される profile は引き続き Node runtime を使う。Python は macOS または Linux にローカルなままであり、設定されたリソース予算とサポートされない呼び出しごとの timeout override は変わらない。

## 移行

1. Python runtime を選択する `cordis.yml` または profile パッチで、既存の共有 sandbox サービスを保持する。それらがない場合は、`@deepseek-ai/dsh-sandbox-local` と `@deepseek-ai/dsh-sandbox-policy` を、必要な `@deepseek-ai/dsh-session-projection` サービスとともにマウントする。[Python runtime の要件](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/ptc-runtime-python/README.md#use-this-package) を参照。
2. agentless 呼び出しのために、共有 `sandbox-policy` 行の `mode` と `workspaceRoot` を設定する。デフォルトモードは `read-only` で、`workspace-write` は workspace への書き込みを許可する。PTC tool 呼び出しは Session の解決済み policy と directory を使う。Python 専用の別個の sandbox 設定を追加しない。
3. 直接のサービス呼び出し元は `runtime.run(runtime.resolve(request))` を使う。明示的な override が必要な場合にのみ、信頼された `sandboxPolicy` を渡す。モデル向けの `run_code` は、1回の呼び出しでより広いモードを使う既存の approval フローを使い、ネストされた tool の権限を変更したり、プログラムを自動的に replay したりしない。
4. プログラムが `workspace-write` の下で意図した workspace 内に書き込め、その外には書き込めず、`result.sandbox.mode` と `result.sandbox.enforcement` を返すことを確認する。`read-only` の下では直接のファイル書き込みが失敗する。`sandbox-unavailable` を拒否されたファイル操作とは別に調べ、再試行前にローカル sandbox インストールを修復する。
