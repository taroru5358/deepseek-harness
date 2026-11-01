# ポストモーテム 0004: Landlock の部分適用通知が子プロセスの失敗を誤分類した

日本語 | [English](../../../docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md) | [中文](../../../docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.zh.md)

> これは非公式の日本語訳です。英語版（[docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md](../../../docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ステータス: 解決済み

## エグゼクティブ・サマリー

古い Landlock ABI を持つカーネルでは、ランチャーが子プロセスを実行する前に無害な部分適用通知を出力する。harness は、この共有された `landlock-run:` プレフィックスと子プロセスの非ゼロ終了の組み合わせをランチャーの失敗として扱ったため、一致なしによる ripgrep の終了コード 1 のような通常の結果が `SANDBOX_UNAVAILABLE` として現れた。当時 bash 経由だったファイルシステム検索も、その構造化エラーを `SEARCH_FAILED` の背後に隠していた。広すぎるシグネチャルールと、部分 ABI 構成のカバレッジ欠如がこの欠陥を通してしまった。ランナーの分類は、正確な情報行の除外を行った上で、ステータスでゲートされた致命的エビデンスを要求するようになった。また、組み立て済みのキー不要シナリオが、生き残った bash 経路を固定する。ファイルシステム検索は subprocess seam 経由でパッケージ済みの ripgrep を使い、サンドボックス化された bash を経由しない。

## 概要

ネイティブランチャーの契約は2種類の stderr 行を区別する。部分的に適用するカーネルは正確に `landlock-run: partial enforcement (older Landlock ABI)` を出力し、子プロセスへ進む。ランチャーの失敗は別の `landlock-run:` 行を出力し、子プロセスを実行せずに終了コード 125 で終了する。

harness は両方を1つの大文字小文字非依存の `landlock-run: ` 部分文字列で表現していた。そのコンシューマーは、その部分文字列を含む非ゼロ終了をすべてランナーの失敗と分類した。したがって子プロセスのステータスがランチャーの情報行に結び付けられた。`false`、一致なしによる ripgrep の終了コード 1、不正パターンによる終了コード 2、さらには子プロセスが選んだ終了コード 125 まで、閉じ込めと実行が成功しているにもかかわらずサンドボックスのせいにされ得た。

インシデント当時、ファイルシステム検索は2つ目の帰属エラーを追加していた。bash 経由の `runRipgrep()` は、中断されなかった拒否された bash 実行をすべて捕捉し、汎用の cwd／シェル起動の `SEARCH_FAILED` に置き換えていた。サンドボックスエグゼキューターが生成した構造化 `SandboxUnavailableError` も含めて。

## 影響

部分 ABI の Landlock ホストでは、正当な子プロセスの非ゼロ結果がサンドボックス基盤の失敗として現れ得た。`glob` と `grep` は特に目立った。ripgrep は空の検索成功に終了コード 1 を使うからだ。ファイルシステム検索を通じて実際のサンドボックス失敗が起きた場合、呼び出し元はその `SANDBOX_UNAVAILABLE` コードを失い、誤った起動診断を受け取った。

この欠陥は閉じ込めを弱めたり、コマンドを非閉じ込めで実行したりしなかった。そのセキュリティ上の影響は可用性と診断の完全性である。有効な閉じ込め済みの結果が拒否されたり、誤ってラベル付けされたりした。

## タイムライン

- ネイティブランチャーの契約は、ランチャー失敗に対して終了コード 125、そのようなすべての失敗に対して致命的な `landlock-run:` 行、子プロセスの実行成功に対して正確な部分適用通知を定義した。
- サンドボックスプロバイダーはその契約を `runnerFailureSignatures: ['landlock-run: ']` に縮約した。bash のコンシューマーはプレフィックスと任意の非ゼロ終了を組み合わせ、stderr の最初の行を報告した。
- ユニットテストは、正常な成功、拒否の診断、致命的なランナープレフィックスを対象としていた。実ランナーのテストは使用可能なカーネルがないと自己スキップし、部分適用の後に非ゼロの子プロセスを続ける状況を強制していなかった。
- 通知を出力してペイロードを `exec` する最小の POSIX ラッパーが、`false` と ripgrep の一致なしで失敗を再現した。
- 構造化ルールと、共有されたフォアグラウンド／バックグラウンド分類、組み立て済みの replay カバレッジが、残っていたサンドボックス帰属の隙間を埋めた。ファイルシステム検索は `ctx.subprocess` 経由でパッケージ済みの ripgrep を使う。修正はその経路をサンドボックス化された bash の外に置いたままにしている。

## 根本原因

公開されているサンドボックスの結果型は、部分文字列の集合しか表現できなかった。Landlock の失敗には終了コード 125 が必要であること、エビデンスは1つの致命的な行の内側に存在しなければならないこと、同じプレフィックスの下の1つの正確な行は情報行であることを述べられなかった。その結果、boolean のコンシューマーは異なるプロセスの無関係な事実を結合し、後の行が致命的エビデンスである場合でも詳細として stderr の最初の行を選んだ。

テストマトリクスもその表現を写していた。フェイクプロバイダーはランナー行を出さないか、明確に致命的なプレフィックスを出した。子プロセスが制御する非ゼロ終了の前に無害なランナー行を出すことはなかった。実際の Landlock カバレッジはホストの ABI に依存していたため、完全 ABI のホストは通知を実行できなかった。インシデント当時の検索実装では、ファイルシステム検索のテストが生の spawn エラーをモデル化していたが、実際のサンドボックス化された bash 構成が throw する構造化エラーはモデル化していなかった。

stderr は依然として帯域内の帰属チャネルである。閉じ込められた子プロセスは、ランナーのゲートされた致命的行と終了ステータスを意図的に再現でき、可用性／診断上の誤帰属を引き起こし得る。より厳密な論理積はこのインシデントの偶発的な衝突を防ぐが、書き手を認証するものではない。帯域外のステータスプロトコルは別個のハードニングであり、サンドボックスバイパスの修正ではない。

## 追加したガードレール

- [`RunnerFailureRule`](../subsystems/sandbox.md#wrapped-argv-and-classification-dialects) は、省略可能な許可終了コード、大文字小文字非依存の行単位の致命的シグネチャ、大文字小文字非依存の正確な情報行の除外を持つ。
- [`dsh-sandbox-local`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/sandbox/sandbox-local) は Landlock を終了コード 125 と通知ではない `landlock-run:` 行にマップする。一方、bwrap、Seatbelt、カスタムランナーはシグネチャのみのままである。
- [`dsh-bash-sandbox`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/bash-sandbox) はプロバイダーの argv を直接 spawn するため、起動前の拒否はローカライズされたシェル診断ではなく spawn エラーのチャネルを使う。確定したフォアグラウンド実行とバックグラウンド実行は、エビデンスを返す1つの分類器を共有する。致命的エビデンスは拒否より優先され、フォアグラウンドのエラーは捕捉済みの stderr を変更せずに一致した致命的行を報告する。
- [`dsh-tool-fs-search`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/fs/tool-fs-search) は `ctx.subprocess` 経由でパッケージ済みの ripgrep を使い、サンドボックス化された bash の seam の外に留まる。
- ネイティブ境界のリグレッションケースは [`partial-landlock.spec.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/bash-sandbox/tests/partial-landlock.spec.ts) にあり、情報通知、致命的エビデンス、フォアグラウンド／バックグラウンド分類を含む。
- 組み立て済みの製品経路は [`partial-landlock` スナップショット構成](https://github.com/deepseek-ai/deepseek-harness/blob/master/snapshots/session/partial-landlock-child-failure/cordis.snapshot.yml) により、ファイルシステム検索の実装上の選択とは独立に固定されている。

## 教訓

- プロセスの帰属には独立したエビデンスの論理積が必要である。共有されたプレフィックスはプロトコルではない。
- 情報診断と致命的診断は名前空間を共有し得るため、除外は正確かつ狭くし、未知の致命的行は fail-closed のままにする必要がある。
- アダプターは、下位の seam が所有する構造化された失敗を、自分に最も近い汎用カテゴリで置き換えるのではなく、保持しなければならない。
- プラットフォーム依存の振る舞いには、ネイティブ境界での決定論的なフェイクと、1つの組み立て済み製品経路が必要である。自己スキップする実カーネルテストだけではそのリグレッションを担えない。
