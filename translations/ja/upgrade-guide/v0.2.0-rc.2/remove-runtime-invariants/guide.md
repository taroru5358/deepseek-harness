---
kind: upgrade-guide
description: "runtime invariant registry パッケージと、各パッケージの `./invariant` subpath export は削除される。"
---

# runtime invariant plugin は削除される

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.2.0-rc.2 では `@deepseek-ai/dsh-invariants` が `ctx.invariants` サービスを提供し、`@deepseek-ai/dsh-session`、`@deepseek-ai/dsh-agent`、`@deepseek-ai/dsh-scope`、`@deepseek-ai/dsh-agent-loop` などの workspace パッケージが、それにチェックを登録する `./invariant` companion plugin を公開していた。`sdk-minimal` profile はこれらの行のうち5つを、id `invariants`、`session-invariant`、`agent-invariant`、`scope-invariant`、`agent-loop-invariant` でマウントしていた。

次のリリースでは `@deepseek-ai/dsh-invariants`、`InvariantRegistry`、`InvariantInstaller`、`InvariantFailure`、`InvariantError`、および `<package>/invariant` subpath を公開しなくなる。`sdk-minimal` profile は5つの行を含まなくなる。`credentials/reference-updated`、`credentials/record-updated`、`authorization/settled`、`llm/adapters-updated` の emitter は、`code: 'INVARIANT'` のエラーを含むすべての listener 失敗を、再 throw する代わりに捕捉してログに記録する。

削除された module を指定する `cordis.yml`、パッチ、overlay はそのロードに失敗する。5つの `sdk-minimal` id のいずれかを対象とするパッチは `patch: entry <id> not found` をログに記録する。削除された module や symbol を import する TypeScript コードはコンパイルされなくなる。

## 移行

1. `cordis.yml`、`$DSH_HOME/profiles/<name>/cordis.patch.yml`、`--patch` overlay から、`name` が `@deepseek-ai/dsh-invariants` であるか `/invariant` で終わる行を削除する。
2. id `invariants`、`session-invariant`、`agent-invariant`、`scope-invariant`、`agent-loop-invariant` を対象とするパッチエントリを削除する。
3. `package.json` の依存関係から `@deepseek-ai/dsh-invariants` を削除し、それと `<package>/invariant` subpath の import を削除する。4つの emitter のいずれかからの再 throw された `INVARIANT` 失敗に依存していた plugin は、独自のチャネルで失敗を報告しなければならない。
4. 確認: `dsh` が module-not-found エラーや `patch: entry ... not found` 警告なしで profile を起動し、`tsc` が欠落 module や export を報告しない。
