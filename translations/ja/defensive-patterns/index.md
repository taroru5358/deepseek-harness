# 防御的パターン

日本語 | [English](../../../docs/defensive-patterns/README.md) | [中文](../../../docs/defensive-patterns/README.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/README.md](../../../docs/defensive-patterns/README.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

苦労して得たルールです。このディレクトリの各パターンは、このリポジトリがレビューで強制する要件であり、ルールとそれが防ぐ欠陥として述べています。すべてのルールは、何が成り立たなければならないか、レビュアーが判定する条件、その境界と例外の承認者、正しい例と誤った例を示します。変更しようとしている成果物のページを、変更する前に読んでください。

## なぜこれらのルールか

5 つの習慣が以下のほとんどをカバーし、各ページはそれらを 1 つの成果物に適用したものです:

- **正しさを明示する。** 後からタイミングで補償するのではなく、構造（名前付きの呼び出しポイント、明示的なライフサイクル、明示的な条件と型）に書き込みます。
- **単一の真実のソースを保つ。** 1 つのデータに 1 つの読み取り経路、1 つの事実に 1 つの置き場所、共有状態に 1 つの所有者。
- **登録されたシームを通じて拡張する。** 新しいケイパビリティは登録し、既存のキャリアは内部に分岐を増やすのではなく、新しいロジックを受け取るのをやめます。
- **変更の範囲をレビューの範囲と等しく保つ。** 変更は、明示した振る舞いの変更が必要とする行だけを運び、移行はコードを書き換えるのではなく移動し、パフォーマンスに敏感な変更はその測定を運びます。
- **エビデンスを要求する。** 主張は反証可能で再計算可能（前後で測定するか、ゲートでチェックする）であり、手作業で同期を保つものではありません。

| 変更するもの | 読むもの |
|---|---|
| パッケージのエクスポート、パッケージ形状、ビルド・フェース、依存関係の宣言 | [package-surface.md](package-surface.md) |
| Remote サービス、`@Remote` メソッド、API ディスパッチ、API レイヤーでのビジネス所有 | [remote-and-api.md](remote-and-api.md) |
| Client コンポーネント、Slot、store、フック、エフェクト | [client-components.md](client-components.md) |
| Session の読み取り、保持、会話ノード、レンダラー | [conversation-and-session.md](conversation-and-session.md) |
| タイミング、状態の表現、共有の所有、分散化、リファクタリング、移行、パフォーマンスのエビデンス、破棄、コールバック、サブプロセス | [general.md](general.md) |

型定義とサブシステムごとのセマンティクスは [subsystems/](../subsystems/index.md) にあります。このディレクトリはルールを運び、セマンティクスのページは型が何を意味するかを運びます。[dsh-code-review](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-code-review/SKILL.md) はプルリクエストを判定するときにこのディレクトリを読み、Development は [AGENTS.md](../../../AGENTS.md) から同じ表を通ります。
