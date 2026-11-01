---
kind: upgrade-guide
description: "`agent-instructions` 行は `dshHome` を受け付けなくなり、instruction のロードは常に `$DSH_HOME` または `~/.dsh` から harness home を解決する。"
---

# instruction のロードは1つの harness home を解決する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-agent-instructions` は `dshHome` 設定フィールドを削除した。plugin は他のすべての harness consumer と同様に、`@deepseek-ai/dsh-home-paths` の `resolveDshHome()` を通じて harness home を解決する。home は `$DSH_HOME` が設定されていて空白でない場合はそれ、そうでなければ `~/.dsh` で、チルダ展開を行う。

`agent-instructions` 行にまだ `dshHome` を設定している `cordis.yml`、profile パッチ、`--patch` overlay は効果がない。行は余分なキーを保持し、plugin はそれを無視するため、instruction のロードはプロセスの home に従う。`dsh-skill-filesystem` や `dsh-shell-env` などの他の provider は独自の home フィールドを保持する。この削除は instruction loader のみを対象とする。

## 移行

1. `cordis.yml`、`$DSH_HOME/cordis.patch.yml`、`$DSH_HOME/profiles/<profile>/cordis.patch.yml`、および `--patch` overlay の `agent-instructions` 行から `dshHome` を削除する。
2. override がデフォルト以外の home を指していた場合は、代わりにプロセス用に `DSH_HOME` をエクスポートし、すべての harness consumer がその home を読むようにする。
3. `dsh --profile <profile> --dump-config` で、`agent-instructions` 行が残りのフィールドのみを列挙すること、新しい session の最初のリクエストがそのファイルの存在時に `Instructions from: $DSH_HOME/AGENTS.md`（または `~/.dsh/AGENTS.md`）を表示することを確認する。
