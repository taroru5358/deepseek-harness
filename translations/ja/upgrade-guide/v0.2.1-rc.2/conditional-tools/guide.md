---
kind: upgrade-guide
description: "tool 宣言は step で admission された登録からのみ行われる。`system-prompt/assemble` は tool を含まなくなり、条件付き tool は解決済みの step セットを必要とし、`exit_plan_mode` は Session が初めて plan モードに入った後にのみ宣言される。"
---

# 条件付き tool を step admission を通じて宣言する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`ctx.tools.register(definition, { when })` は同期または非同期の条件を追加する。AgentLoop は提案された step ごとに `{ agent, modelSelection, signal }` で条件を1回解決し、prompt assembly と `agent/pre-step` の前に、その step と再試行における tool 宣言、Native dispatch、PTC binding、ネストされた呼び出しに結果の `ResolvedToolSet` を使う。条件が false の場合はそれらすべてから tool が削除され、条件が throw した場合は `TOOL_ACTIVATION_FAILED` で turn が終了する。

`system-prompt/assemble` waterfall は tool 配列なしで section、context、variable（`PromptPresentation`）を受け取るようになり、返された `tools` フィールドは無視される。その waterfall で tool 宣言を削除または追加していた plugin が影響を受ける。削除はリクエストに届かなくなり、`assembly.tools` を読む listener は `undefined` を受け取って runtime で失敗する。

モデルの tool 呼び出しは、その step で admission されたセットを通じてのみ実行される。解決済みセットのない条件付き tool への呼び出し、セットが破棄された後の呼び出し、step 中に登録が置き換えられた、または削除された pending 呼び出しは、実行されずに `UNKNOWN_TOOL` を返す。置き換えはすべての tool に適用されるため、hot reload、MCP reconnect、skill reload が step の途中で tool を再登録すると、その step の未開始の呼び出しが失敗する。実験的な `claude-code-mods` の `$.tool.call` のようにセットを渡さない呼び出し元は、条件付き tool を呼べない。

`exit_plan_mode` は、選択された plan 状態が active である最初の step から宣言され、Session の残りでも宣言されたままになる。そのため、plan モードに入ったことのない Session のリクエストにはそれが列挙されなくなった。既存の Session に形式の移行は不要である。

## 移行

1. `system-prompt/assemble` listener での tool フィルタリングと `assembly.tools` の読み取りを、登録の `when` 条件に置き換える。step の provider、model、reasoning effort は条件の `modelSelection` から読む。waterfall には section、context、variable の変更のみを残す。
2. tool guidance も同じ admission に従わせる。section callback は `context.toolSet` を `ctx.tools.get()` または `ctx.tools.schemas()` に渡す。
3. 条件付き tool の programmatic 呼び出しでは `ctx.tools.resolve({ agent, modelSelection, signal })` を呼び、返された `toolSet` を `assemble()` と `ctx.tools.execute()` バッチに渡し、バッチの settlement 後に破棄する。共有 pipeline を使わないカスタム executor は、各呼び出しを `ctx.tools.executable(name, agent, toolSet)` で lookup する。
4. Session が plan モードに入る前に `exit_plan_mode` を列挙していた期待を更新する。無効な tool がリクエストに存在しないこと、その名前を呼ぶと実装に到達せず `UNKNOWN_TOOL` を返すこと、失敗する条件が `TOOL_ACTIVATION_FAILED` で turn を終了することを確認する。
