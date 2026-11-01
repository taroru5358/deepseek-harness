# クックブック: プラグイン機能プレビューの追加

日本語 | [English](../../../docs/cookbook/adding-a-plugin-preview.md) | [中文](../../../docs/cookbook/adding-a-plugin-preview.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-plugin-preview.md](../../../docs/cookbook/adding-a-plugin-preview.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

機能プレビューは、バンドルの詳細ドキュメント内の任意のイラストです。詳細ドキュメントは、**Plugins** 内のそのページが説明の下に表示するドキュメントです。公式バンドル用に描く場合はこのガイドに従ってください。詳細ドキュメント自体は [`meta.detail`](adding-a-package.md#plugin-display-metadata) を通じて宣言します。

## 1. 画像が役立つか決める

具体的な結果を1つ、または認識可能なケイパビリティを示します。説明で十分なら画像は省略します。画像は必須ではありません。必須の説明はバンドルの説明内、またはイラストの外の設定に残します。

## 2. 固定サイズの SVG を1つ描く

| 項目 | 要件 |
|---|---|
| 枚数 | バンドルごとに0枚または1枚の機能画像。英語と中国語のファイルはその画像の代替であり、ギャラリーやカルーセルではない。 |
| 形式 | SVG: `width="640" height="300" viewBox="0 0 640 300"`（32:15）。 |
| 言語 | 英語と中国語の両方のファイルを提供する。見出し、注釈、例の内容を翻訳し、公式ブランド資産と名前は保持する。 |
| 構成 | 主要な例を1つ、任意でクローズアップを1つ。重要なコンテンツは端から約32キャンバス単位離す。テキストはページの実際のサイズでレビューする。 |
| タイポグラフィ | 適切なフォントを使って文字をパスにアウトライン化する。機能と隣接 UI に合わせ、ブラウザのフォント置換に頼らない。 |
| テーマ | ライトとダークのテーマを比較する。1つのイラストが両方にサービスする。ダーク背景で消える透明な暗いテキストは避ける。 |

SVG は自己完結しています。フォント、リモート・リソース、スクリプト、ハンドラー、アニメーション、スタイルシート、`text`、`foreignObject` はありません。`image` 要素は、変更されていない公式 PNG/JPEG/WebP ブランド資産を base64 data URL として埋め込んでもかまいません。小さなプラグイン・アイコンはジオメトリのみのままです。ページは SVG を `<img>` でレンダリングします。各ファイルを小さく保ってください。Host は plugin-manager の `detailImageMaxBytes`（既定 5 MiB）を超える画像を拒否します。

## 3. 詳細ドキュメントにする

ペアをバンドル・パッケージ内の `docs/preview.svg` と `docs/preview.zh.svg` として、`docs/DETAIL.md` と `docs/DETAIL.zh.md` の隣に保存し、画像を各ドキュメントの唯一の行にします:

```markdown
![Example feature preview](preview.svg)
```

中国語ドキュメントは中国語の代替テキストで `preview.zh.svg` を参照します。[バッジ・スキル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/badge-skill-bundle/docs/DETAIL.md)と[音声入力](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/voice-input-bundle/docs/DETAIL.md)のドキュメントを例として使ってください。音声トランスクリプトは説明用のサンプル内容であり、録音結果ではありません。バンドルが何をし、どのように使うかを振る舞いのレベルで `meta.description` に記述します。ページはそれを画像の上に表示します。

## 4. ページの振る舞い

| 項目 | 現在の振る舞い |
|---|---|
| 位置 | 詳細ドキュメントの先頭。詳細ドキュメントはバンドル・ページの最初のセクションで、説明とステータス通知の下にある。 |
| 表示 | 詳細ドキュメントは画像を自身の幅までで、高さが制限されたスクロール領域内にレンダリングする。 |
| 言語 | UI 言語のドキュメントが自身のファイルを参照する。 |
| 可用性 | 有効化前に表示される。リストとコンポーネント行は小さなアイコンを保つ。 |

## 5. 検証とレビュー

`scripts/plugin-preview-assets.spec.ts` はバンドルの `docs` ディレクトリ内のすべてのプレビュー・ペアをチェックします。固定キャンバス、スクリプト、スタイルシート、テキスト要素、外部参照の不在、1 MiB のペア上限、英語と中国語のファイルが異なることを確認します。その後、有効化前に実際のプラグイン詳細を英語と中国語で開き、画像が読み込まれること、配置、ライト／ダークでの可読性、390 px で水平オーバーフローがないことを確認します。視覚品質と両言語での等価な意味はレビューが所有します。PR 証拠には [record-browser-gif](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/record-browser-gif/SKILL.md) に従ってください。最終的なクリーン・コミットから記録し、実演するフローを正確にラベル付けします。
