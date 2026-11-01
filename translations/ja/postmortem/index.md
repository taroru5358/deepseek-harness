# ポストモーテム

日本語 | [English](../../../docs/postmortem/README.md) | [中文](../../../docs/postmortem/README.zh.md)

> これは非公式の日本語訳です。英語版（[docs/postmortem/README.md](../../../docs/postmortem/README.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

インシデントの記録: バグが本来到達すべきでない場所（実際のユーザー、マージ済みの PR、リリース）に到達してしまった事例を扱う。興味深いのは *なぜ我々のプロセスがそれを通してしまったのか* であり、1行の修正だけではない。

ポストモーテムは [Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/README.md) ではない（Agent Note は意図的な設計判断と却下した代替案を記録するか、将来の作業を提案する）。ポストモーテムは失敗を後ろ向きに記録するものだ。何が壊れたか、そのメカニズム、なぜすべての安全網がそれを見逃したか、そして同じ種類のバグが次回は目に見える形で失敗するように追加した具体的なガードレール。

次の3条件を満たすときに書く。バグが **subtle**（メカニズムが自明でなく、注意深いエンジニアでも苦労して再導出することになる）、**systemic**（見逃した理由がテスト・ツール・規約の隙間であり、単発のタイポではない）、**costly to rediscover**（実際のデバッグ時間を費やし、今後も費やすことになる）。ポストモーテムの動機になったガードレール（テスト、AGENTS.md のルール、ADR）へのリンクを張る。

すべてのポストモーテムは **Executive summary** で始まる。忙しい読者が30秒で把握できる短い1段落で、何が壊れたか、平易な言葉での根本原因、なぜ見逃したか、そして長く残る教訓を示す。その後に詳細な Summary / Timeline / Root cause / Guardrails の各セクションが続く。

| # | タイトル |
|---|---|
| [0001](0001-acp-default-export-drops-inject.md) | 接続時に ACP サーバーがクラッシュ: `export default` がプラグインの `inject` を落とした |
| [0002](0002-js-expression-disabled-filesystem-tools.md) | リテラルの `!!js` オブジェクトによりファイルシステムのスナップショットツールが恒久的に無効化された |
| [0003](0003-web-agent-gui-feedback-loop.md) | Web agent がセッションをホストする GUI ではなく代替サーバーを検証した |
| [0004](0004-landlock-partial-notice-misclassified-child-failures.md) | Landlock の部分適用通知が子プロセスの失敗を誤分類した |
