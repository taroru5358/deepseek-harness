# Web UI スタイル・リファレンス

日本語 | [English](../../docs/web-styling.md) | [中文](../../docs/web-styling.zh.md)

> これは非公式の日本語訳です。英語版（[docs/web-styling.md](../../docs/web-styling.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このリファレンスは、ブラウザ・クライアント・パッケージのスタイリングの所有関係とコンポーネントのルールを定義します。現在のトークン値は [`packages/client/ui-theme/src/styles/`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/ui-theme/src/styles) にあります。この文書は、ソースから生成されるその一覧を複製しません。

## 所有

[`ui-theme`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-theme/README.md) は `--dsw-*` の静的スケール、セマンティック・エイリアス、タイポグラフィ、モーション、グラデーション、シャドウ、スクロールバーのスタイル、ライト／ダークの設定を所有します。[`ui-layout`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-layout/README.md) は解決済みのテーマ・スナップショットをドキュメントに適用します。機能パッケージはセマンティック・エイリアスを消費し、別のグローバル・テーマを定義しません。

グローバルなスタイルシートは `ui-theme/src/styles/` に置きます。コンポーネントのスタイルは、そのコンポーネントの隣に CSS Modules として置きます。コンポーネントは、値がそのコンポーネントのレイアウトや表示のコントラクトの一部である場合にローカルのカスタム・プロパティを定義できます。共有される色、タイポグラフィ、エレベーション、モーションはテーマ・パッケージに属します。

<a id="component-rules"></a>
## コンポーネントのルール

- コントロールを作り直す前に再利用します。[ui-primitives のコンポーネント・カタログ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-primitives/README.md#component-catalog)は機能パッケージをまたぐ唯一の経路であり、意図的な見た目の違いは2つ目のコピーではなく、そこの prop に置きます（[リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-primitives/README.md)）。
- CSS Modules と `clsx` を使います。コンポーネント・ライブラリや Tailwind を追加してはいけません。
- 機能コンポーネントでは `--dsw-alias-*` のセマンティック・トークンを使います。静的なパレット値をコピーしたり、リテラルの色を書いたりしてはいけません。
- 機能コンポーネントの CSS にテーマ・セレクターを持ち込まないでください。ライト／ダークの上書きはテーマの所有者に属します。
- フォント・サイズには行の高さを組み合わせ、既存のロールが一致する場合はテーマのタイポグラフィ変数を使います。
- ソース・テキスト、ターミナル出力、diff の行は、コンポーネントのコントラクトが桁の保持を要求する場合は折り返さずに保ちます。コンポーネント固有のスクロールバー・セレクターではなく、共有のスクロールバー・スタイルを使います。
- 表示は CSS に置きます。React のインライン・スタイルはコンポーネント・ローカルのカスタム・プロパティ値を渡してかまいませんが、テーマの分岐をエンコードしてはいけません。
- プラグインのスタイルシートにある相対 `url()` は、パッケージの `src/client/assets/` 内のファイルを指し、ビルドがそれを[パッケージ・アセット](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md#dependency-declaration)の URL に書き換えます。`data:`、`#fragment`、`/` 起点、スキーム付き、空の `url('')` 参照はそのまま残り、フラグメントやクエリを伴う相対ファイル参照はビルドに失敗し、依存関係のスタイルシートは書き換えられません。
- `:hover` セレクターは、ビューポート幅とは独立に `@media (hover: hover)` の下に置きます。選択済み、展開済み、押下済み、キーボード・フォーカスのスタイルはその条件の外に保ちます。ホバーがない場合でも、アクションは明示的なコントロールを通じて表示されるかアクセス可能なままです。ポインター・ハンドラーは、ホバー専用のプレビューを開くときにタッチを無視します。キーボード・フォーカスの可視性と reduced-motion の挙動は保ちます。
- 角丸は、対応エンジンでは ui-theme の `corner-shape.css` によるグローバルなスーパー楕円のスムージングを継承します。完全な丸めの `border-radius`（`50%`、`100%`、ピル半径）には必ず `corner-shape: round` を組み合わせ、円とカプセルが円弧を保つようにします。ui-theme の corner-shape の spec がこの組み合わせを強制します。
- 浮き上がったサーフェス（メニュー、ポップオーバー、モーダル、パネル、フローティング・ボタン、コンポーザー）は `border: 0` を設定し、`box-shadow: var(--dsw-elevation-panel)`、`var(--dsw-elevation-prominent)`、またはコンポーザーの `var(--dsw-elevation-soft)`（より低いアルファでより大きなブラー）を使います。0.5px のヘアライン・ストロークが最初のシャドウ層であり、`--dsw-elevation-stroke-color` がサーフェスや状態ごとにそれを再バインドまたは抑制します。`--dsw-alias-border-*` のボーダーと lv／エレベーションのシャドウを組み合わせてはいけません。ui-theme の elevation の spec がこの組み合わせを拒否します。状態色のボーダー（警告パネル）は実際のボーダーのままです。
- 既存および新規のドロップダウン、コンテキスト、サブメニュー、選択メニューは `Menu` を使うか、カスタム・コンテンツを `MenuSurface` でラップします。素材はテーマが所有する `--dsw-menu-surface-fill` と `--dsw-menu-backdrop-filter` です。機能側やプラットフォーム側の CSS はどちらも上書きしてはいけません。ui-theme のメニューの spec は、ラップされていないメニュー／リストボックスのレンダラー、素材の上書き、トークンの再定義を拒否します。schedule が所有する `TaskMenu` と `ClockPicker` は、明示的な例外として既存のコンテナと素材を保ちます（[所有者](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-schedule/README.md)）。
- モーダルの背景（設定ダイアログや確認ダイアログを含む）は、ブラーなしの半透明の暗いマスクを保ちます。テーマは `--dsw-mask-blur` を `none` に設定します。Desktop が所有するモーダルのオーバーレイも、親ページをフィルターしません。
- メニュー背景のフィルタリングは分離されたレイヤーに留め、ネストしたメニューや固定オーバーレイが位置と背景を保つようにします。macOS では、`MenuSurface` が各メニューの境界内だけページ・コンテンツの背後に不透明な下地を追加し、Chromium がネイティブの vibrancy の上でコンテンツをぼかせるようにします。CSS のアンカーは配置とサイズに従い、アンマウントすると下地は取り除かれます。この下地を持たないオーバーレイは `--dsw-specific-menu` を使います。テーマが所有する macOS 向けの塗りは 94% 不透明で、下のテキストが透けないようにします。この塗りを使う他の浮き上がったサーフェスは、固定オーバーレイを含む場合、自身の分離した背景に同じフィルターを組み合わせます。
- 中立の `--dsw-alias-border-*` トークンを使うフラットなボーダーと区切り線は `0.5px` で描きます。ボタン、入力欄、カード、行の区切り、塗りつぶしのボックスとして描く区切り（メニューの区切り、会話ヘッダーの継ぎ目、Markdown の `hr`、縦のレール）は同じヘアラインの太さを共有し、Chromium はこれを1デバイス・ピクセルとして描画します。破線のアフォーダンスと状態色のボーダーは 1px のままです。スピナーのリングのトラックは spec の明示的な許可リストを通じて幅を保ちます。ui-theme の elevation の spec は、より広い中立の実線ボーダーを拒否します。
- クリック可能なアーティファクト・リンク（Markdown のアンカー、本文中のファイル言及、Web ソースと fetch のリンク、生成ファイルのチップ、ワークフロー・メンバーのリンク）は `--dsw-alias-link` で `font-weight: 500` にし、通常時は下線なし、ホバー／フォーカス時は 3px オフセットの点線下線を付けます。コンパクトな Thinking Markdown は三次テキスト色と通常時の点線下線を保ちます（[コンパクト表示](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-primitives/README.md#rendering-agent-output)）。テキスト先頭のアンカーは、ui-primitives の `LinkIconMedium` カテゴリ・グリフを `currentColor` で先頭に付けます。よく知られた外部ホストでは地球のアイコンではなくそのサイト自身のマークになり、ワークフロー・メンバーのリンクと画像のみのアンカーはグリフを持たず、ツール行のファイル・リンクは灰色の点線アフォーダンスを保ちます。

<a id="input-capabilities"></a>
## 入力ケイパビリティ

`pointer` と `hover` はブラウザが選んだ主ポインターを表し、`any-hover` は利用可能なすべてのポインターを考慮します。どれも物理キーボードを検出しません。

- ポインターの精度に依存する表示（タッチ向けのサイズのコントロールやテキスト・フィールドなど）には `(pointer: coarse)` を使います。
- CSS のホバー効果は `(hover: hover)` の下に保ち、タッチ優先のレイアウトがホバーなしでも使えるようにします。
- JavaScript のホバー・プレビューは、マウスの進入を受け付けるときに `(any-hover: hover)` を使い、タッチの進入を無視します。そのため、タッチスクリーンが主ポインターのままでも、セカンダリのマウスがプレビューを開けます。`any-hover` で CSS のホバー・スタイルをすべて有効にすると、タッチで生成される `:hover` 状態にも有効になってしまいます。

入力手段が混在するデバイスでは、主ポインター向けの CSS ホバー・スタイルが無効なまま、マウスのプレビューが現れることがあります。キーボード・フォーカスとショートカットのヒントは、これらのケイパビリティ・クエリとは独立に、実際のキーボード操作に従います。[メニューと入力モダリティ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-primitives/README.md#input-modality)が共有の挙動を定義します。

<a id="corner-radii-and-settings-cards"></a>
## 角丸と設定カード

[DSH の統一角丸標準](ui-radius.md)が、半径スケール、コンポーネントのサイズ対応、円とカプセルの例外、ネストしたホバーの形状、設定カードの塗りとストロークを定義します。

## システムを変更する

共有トークンは、所有する `ui-theme` のシートに追加または変更し、機能パッケージからそのセマンティック・エイリアスを消費します。公開のスタイリング・コントラクトが変わった場合は、所有パッケージのリファレンスを更新します。視覚的な挙動は[テスト方針](testing.md)に従います。[アーカイブ済みのスタイリング・システムの Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/process/2026-07-19-web-styling-system.md)がフレームワークの根拠を記録しています。
