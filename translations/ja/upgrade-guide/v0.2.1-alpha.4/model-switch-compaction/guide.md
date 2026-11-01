---
kind: upgrade-guide
description: "自動モデル切り替えには、有効な移行先の圧力予算と成功する compaction が必要である。"
---

# モデル切り替えは移行先の context 圧力を確認する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`compaction-basic` の自動モードが有効な場合、provider/model の切り替えは、新しいリクエストヘッダーを記録する前に移行先の圧力を確認するようになった。以前は、proactive な圧力チェックは最後にコミットされた route を使い、設定または要約の失敗時に警告していた。切り替えは、移行先の容量を解決できない場合、その圧力予算が無効な場合、または compaction が履歴をしきい値未満に減らせない場合に `MODEL_SWITCH_COMPACTION_FAILED` で失敗するようになった。これは、移行先の予算が無効な短い会話にも影響する。完了した pruning と以前の checkpoint は残り、以前のリクエストヘッダーは保持される。

## 移行

1. 選択可能なすべての route に、`resolveModelInfo()` が `context.contextWindow` を宣言する登録済み adapter があることを確認する。このメタデータを持たない middleware 所有の route では、adapter を通じて容量を公開するか、手動のみの compaction に `auto: false` を使う。これは自動の圧力回復と overflow 回復も無効にする。

2. 小さなウィンドウでは、正確な provider/model policy の下で `cordis.yml` に `headroomTokens` を設定する。デフォルトは 65,536 トークンで、route されたリクエストの出力予約と共存できなければならない。65,536 トークンのウィンドウと 4,096 トークンのリクエスト出力上限では、この policy は正の圧力予算を残す:

   ```yaml
   - name: '@deepseek-ai/dsh-compaction-basic'
     config:
       modelPolicies:
         - provider: local
           model: small-context
           headroomTokens: 4096
           maxTokens: 4096
   ```

3. `local` と `small-context` を登録済みの route に置き換える。policy の `maxTokens` は要約出力を上限し、移行先リクエスト自身の出力上限は別である。解決された保持予算をトリガー未満に保つ。短い履歴で一度切り替えて設定を検証し、次に長い履歴で切り替えて凝縮を検証する。切り替えが失敗した場合は、報告された原因を修正して再試行するか、より大きなモデルを選ぶ。[パッケージ設定リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction-basic/README.md#use-this-package) を参照。
