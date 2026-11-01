# モデルを設定する

日本語 | [English](../../../../docs/user/guide/providers.md) | [中文](../../../../docs/user/guide/providers.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/guide/providers.md](../../../../docs/user/guide/providers.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このガイドは、[ルート README](../../overview.md#run) から Web UI を起動済みであることを前提とします。モデルの変更は、サーバーを再起動せずに次のリクエストから反映されます。

## DeepSeek を設定する

**Settings → Models** を開きます。DeepSeek カードには API キーの入力欄が1つあり、キーを入力して保存します。

![モデルページ: DeepSeek カードと、その下の Add model provider](../../../../docs/user/guide/providers-models-page.png)

キーは書き込み専用です。保存後にページが受け取るのは伏字化された記述子であり、秘密の値そのものではありません。キーは `$DSH_HOME/.credentials.yaml` に保存され、設定側にはそのクレデンシャル参照だけが残ります。

キーを保存し、既定のエンドポイントを使う場合、DeepSeek のモデルは DeepSeek API が公開する一覧に従います。一覧はモデルが最初に必要になったときに読み込まれ、使用中は約10分ごとにバックグラウンドで更新され、読み込めない場合は組み込みモデルにフォールバックします。**Customized settings** の **Models** を編集すると、独自の一覧を保持して公開一覧への追従を停止します。**Restore defaults** を選ぶと再び追従し、一覧を再読み込みします。公開一覧の使用中、カードには組み込みモデルが表示されるため、新しく公開されたモデルはモデルピッカーにのみ現れます。一覧取得、フォールバック、ゲーティングの規則は [API キー provider リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek-api-key/README.md) が管理します。

## サードパーティ provider を追加する

**Add model provider** を選択します。カードは **Third-party model provider** で開きます。dsh に同梱された provider を選び（一覧には `anthropic`、`openai`、Kimi 用の `moonshotai`、GLM 用の `zai` などの provider id が表示されます）、API キーを入力して保存します。インストール済みカタログがエンドポイント、プロトコル、モデル一覧を提供します。

認証の **?** は、保存済みクレデンシャルの優先順位と環境変数フォールバックを説明します。キーを空欄にすると、保存済みクレデンシャルを使うか、一覧の環境変数を試します。緑のヒントは環境変数が検出されたことを示します。黄色のヒントはクレデンシャル変数が検出されなかったことを意味し、API キーを入力するか一覧の変数を設定してください。未確認のヒントはステータスの読み取りに失敗したことを意味します。検出はクレデンシャルの検証や接続テストを行いません。AWS プロファイルや Google のアプリケーション既定クレデンシャルなど、provider が管理するクレデンシャルファイルはこのチェックの対象外です。`openai-codex` のようにネイティブのキー認証を持たない provider では、profile がクレデンシャル参照を明示しない限りキー欄は表示されません。

保存済みクレデンシャルを使う場合は、プレースホルダー **Leave blank to use saved credentials** が表示され、入力欄の下にステータステキストは出ません。環境変数フォールバックでは **Leave blank to use env as auth** と、入力が空の間は検出ヒントが1つ表示されます。入力が空でなければ認証ステータスのテキストは表示されません。アドレス設定用の変数は **Base URL** の下に表示され、クレデンシャルとしては数えられません。

キー入力が空でない場合は目のボタンがあります。**Show secret** を選ぶと入力内容を確認でき、**Hide secret** で再びマスクされます。入力を変更すると再びマスクされます。保存済みクレデンシャルは書き込み専用のままです。

## カスタムモデル API を追加する

中継サーバー、社内ゲートウェイ、セルフホストサーバー、またはインストール済みカタログにない provider には、カードを **Custom model API** に切り替えます。小文字の Provider ID、ベース URL、API プロトコル、クレデンシャル、少なくとも1つのモデルを指定します。**API protocol** はゲートウェイが話すプロトコルでなければならず、ピッカーには OpenAI Chat Completions、OpenAI Responses、Anthropic Messages の3つが用意され、アクティブな profile の `cordis.patch.yml` にそれぞれ `openai-completions`、`openai-responses`、`anthropic-messages` として保存されます。1つの provider は1つのプロトコルを話すため、2つのプロトコルを提供するゲートウェイには2つの provider が必要です。

![カスタムモデル API フォーム: Provider ID、表示名、ベース URL、API プロトコル、API キー](../../../../docs/user/guide/providers-custom-form.png)

Provider ID は、リクエスト、保存済み session、モデルの既定値、クレデンシャル参照がそれを使うため永続的です。provider の名前を変更するには、新しい provider を追加して古いものを削除します。表示名、ベース URL、プロトコル、クレデンシャル、モデルは編集可能なままです。

### モデルを検出する

**Model catalog** の下で **Fetch available models** を選ぶと、エンドポイントに提供しているモデルを問い合わせます。リクエストはフォーム内の現在のベース URL、プロトコル、キー、または保存済み provider の保存キーを使い、応答は検索可能なピッカーで開きます。一覧に既にあるモデルはチェック済みで表示されるので、検索して追加するモデルにチェックを入れ、削除するモデルのチェックを外し、**Update list** を選びます。エンドポイントが一覧に載せない設定済みモデルは残ります。provider を保存または作成するまで何も保存されません。

検出は一般的なゲートウェイが公開する一覧形式を読みますが、すべてのエンドポイントがそのいずれかで応答するわけではないため、保証ではなく便宜的な機能と考えてください。失敗したり何も一覧に載らなかったりした場合は、モデル id を手動で追加すれば同じように動作します。組み込み provider は、ベース URL がゲートウェイを指していても常にインストール済みカタログから応答されるため、ゲートウェイが実際に提供するものを確認するにはカスタム provider 経由で取得してください。

## モデルを選択する

設定済みの provider はモデルピッカーに表示されます。モデルを選ぶと、新しい session の既定値にもなります。すでにリクエストを送信した session は、自身のログに記録されたモデルを保持します。

provider のモデル順序を変更するには、**Settings → Models** で編集し、**Customized settings** を開き、モデルの3本線ハンドルをドラッグします。ハンドルにフォーカスして上下の矢印キーを押すこともできます。**Apply** で順序を保持し、**Cancel** で破棄します。モデルピッカーは、その provider 内で保存された順序に従います。

保存済みの既定値が削除済みの provider を指している場合、composer は **Select model** を表示し、別のモデルが選択されるまで入力をブロックします。

## 詳細設定

生成された [plugin 設定カタログ](../../../../docs/config-catalog.md) には、すべての plugin でサポートされる全フィールドと既定値が掲載されています。[`dsh-llm-pi-ai`](../../../../docs/config-catalog.md#deepseek-aidsh-llm-pi-ai) は、このページで設定する provider セクションです。[`dsh-llm-pi-ai`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md) と [`dsh-llm-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek/README.md) のリファレンスは、`cordis.patch.yml` の直接設定、カタログ解決、reasoning 制御、クレデンシャル、アダプターのエラーを扱います。

::: tip 追加設定
Models ページでは、API キー、表示名、ベース URL、API プロトコル、および各モデルの id、表示名、コンテキストウィンドウ、最大出力トークン、入力タイプに加え、カスタムモデル API provider のモデルの reasoning レベルを公開します。組み込み provider のモデルの reasoning レベル、リクエスト互換性スイッチ、ヘッダー、タイムアウト、リトライポリシーは、ページが書き込むのと同じ文書である `$DSH_HOME/profiles/<profile>/cordis.patch.yml` で設定します。直接編集するか、ブラウザがサーバーと同じマシンで動いている場合は Settings ヘッダーの **Open configuration file** で開きます。アダプターは次のリクエストで再読み込みするため、再起動は不要です。以下のサブセクションでは、ほとんどのゲートウェイが必要とするフィールドを説明します。

`dsh web` による標準の Web UI 起動では `<profile>` は `web` なので、パスは `$DSH_HOME/profiles/web/cordis.patch.yml` です。カスタム profile を起動する場合は、起動時に選択した名前を使ってください。
:::

### 画像入力

**Settings → Models** で provider を編集し、**Customized settings** を開き、モデルの **Model options** を展開します。**Input types** は容量フィールドの下に独立した行として表示されます。画像を受け付けるモデルでは **Image** を選択して保存します。新規のカスタムモデルでは、継承された画像 capability がない場合、**Text** が選択された状態で始まります。少なくとも1つのタイプを選択したままにする必要があり、画像専用モデルにするには Text を外す前に Image を選択します。

チェックボックスは、pi-ai モデルでは `input` を、直接の DeepSeek アダプターでは `inputModalities` を保存します。`$DSH_HOME/profiles/<profile>/cordis.patch.yml` でモデルを直接編集することもできます。たとえば、次のカスタム pi-ai provider はテキスト専用モデル1つと vision モデル1つを宣言しています:

これらの例は profile パッチ内の設定フィールドを示しています。Cordis の設定オーバーライドはエントリの設定全体を置き換えるため、既存のオーバーライドを編集するときは他の provider とフィールドを保持してください。

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-gateway:
        apiKeyEnv: GATEWAY_API_KEY
        api: openai-completions
        baseURL: https://gateway.example/v1
        models:
          - id: legacy-chat
          - id: vision-preview
            input: [text, image]
```

Pi-ai の `input` は `text` と `image` を受け付け、そのモデルだけに適用されます。明示的な空でない選択が優先されます。`input` を省略するか空にすると、インストール済みカタログの入力タイプを継承し、次にルートの `defaultInput`（既定は `[text]`）を継承します。チェックボックスは、行を開いただけではオーバーライドを保存せず、これらの継承値を表示します。

DeepSeek は `inputModalities` の省略をテキスト専用とみなし、空のリストを拒否します。Image を外すと、そのモデルの `imagePixelBudget` と `imageMaxBytes` も削除されます。DeepSeek はテキスト専用モデルでの画像制限を拒否するためです。後で画像を有効にしてカスタム制限が必要になった場合は、これらの制限を再度設定してください。

チェックボックスを編集した後に継承に戻すには、`cordis.patch.yml` からモデルの `input` または `inputModalities` フィールドを削除します。**Restore defaults** は他のモデル編集を含むモデルカタログのオーバーライド全体を削除するため、カタログ全体を復元したい場合にのみ使ってください。

手動で入力したすべてのモデルが画像を受け付ける場合は、各モデルではなくルートで一度フォールバックを設定します:

```yaml
- id: llm-pi-ai
  config:
    providers:
      vision-gateway:
        apiKeyEnv: GATEWAY_API_KEY
        api: openai-completions
        baseURL: https://vision.example/v1
        defaultInput: [text, image]
        models:
          - id: first-model
          - id: second-model
```

`defaultInput` はオーバーライドではなくフォールバックで、既定は `[text]` です。組み込み provider では、カタログが記述していないモデルにのみ応答するため、画像を持つカタログモデルから画像を外すことはありません。そのようなモデルを狭めるには、そのモデル自身の `input` を使います。組み込み provider に明示的な `models` リストがない場合は、モデル id をキーにして `modelOverrides` の下に書きます:

```yaml
- id: llm-pi-ai
  config:
    providers:
      anthropic:
        modelOverrides:
          claude-sonnet-4-5:
            input: [text]
```

pi-ai の設定では、モデル自身の `input` を除き、すべてのリストは少なくとも1つのモダリティを指定する必要があります。`input` では空のリストは省略と同じ意味です。未知のモダリティは、どこに書かれても拒否されます。

どちらのフィールドも、エンドポイントについての主張を述べるだけで、チェックは行いません。エンドポイントが提供しない画像を宣言したモデルはここでは検出されず、代わりに provider がリクエストを拒否します。

### DeepSeek アカウントのモデル

サインイン済みの DeepSeek アカウントはアカウントサービスからモデル一覧を取得するため、Models ページにはそのモデルエディターがありません。単一のモデルを調整するには `$DSH_HOME/profiles/<profile>/cordis.patch.yml` の `modelOverrides` を使います。フィールドは DeepSeek の `models` エントリと一致し、設定しないフィールドは一覧の値を保持します:

```yaml
- id: llm-deepseek-account
  config:
    modelOverrides:
      deepseek-flash:
        contextWindow: 524288
        maxTokens: 131072
```

より新しい DeepSeek Harness を必要とするモデルは `CLIENT_UPGRADE_REQUIRED` で失敗し、インストールすべきバージョンを示します。一覧取得、フォールバック、ゲーティングの規則は [アカウント provider リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek-account/README.md) が管理します。

### 推論の強度

モデルピッカーは、reasoning レベルを宣言したモデルに対して **Effort** メニューを表示します。組み込み provider のモデルはインストール済みカタログからレベルを継承し、DeepSeek のモデルはアダプターのレベルを保持します。手動で入力したモデルはレベルを追加するまで何も宣言しないため、それまではメニューに Effort の項目が現れず、モデルが推論するかどうかはエンドポイント側の既定値で決まります。

カスタムモデル API provider では、**Settings → Models** でレベルを宣言します。provider を編集し、**Customized settings** を開き、モデルの **Model options** を展開して **Reasoning levels** をオンにします。行は off、low、medium、high で始まります。各行はレベルと、そのレベルに対して送信される **API value** を組み合わせており、API value の既定値はレベル名です。モデル provider の API ドキュメントがレベルの綴りを変えている場合にのみ変更してください。エンドポイントがサポートする内容に合わせて行を追加または削除し、off 以外のレベルを少なくとも1つ残して保存します。スイッチをオフにすると宣言が削除されます。

ページは `$DSH_HOME/profiles/<profile>/cordis.patch.yml` に `reasoningEfforts` を書き込みます。そこではレベルを直接宣言することもできます:

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-gateway:
        apiKeyEnv: GATEWAY_API_KEY
        api: openai-completions
        baseURL: https://gateway.example/v1
        reasoning: high
        models:
          - id: my-reasoner
            reasoningEfforts:
              off:
              high: high
              max: max
```

各キーはメニューが提供するレベルで、その値は `reasoning_effort` としてワイヤー上で送信される綴りです。したがって `max: xhigh` は、独自の語彙を持つゲートウェイ向けにレベル名を変更します。空のままでよいのは `off` だけです。ほとんどのエンドポイントでは、推論しないことはパラメーターの不在だからです。ルートの `reasoning` は、session が何も選んでいないときに使われるレベルです。ピッカーで effort を選ぶと、そのモデルとともに新しい session の既定値として保存されます。

空の `off` は何も送信せず、これはリクエスト時に推論するモデルを止めるだけです。`off` に値を与えると、代わりにその値が `reasoning_effort` として送信されます。指示されない限り推論するモデル（たとえば OpenAI 互換ゲートウェイ背後の DeepSeek V4）には `compat.thinkingFormat: deepseek` が必要です。これにより `off` は `thinking: {type: disabled}` を送り、他のすべてのレベルは effort と並べて `thinking: {type: enabled}` を送ります:

```yaml
        models:
          - id: deepseek-v4-pro
            compat:
              thinkingFormat: deepseek
            reasoningEfforts:
              off:
              high: high
              max: max
```

ゲートウェイが推論しない組み込み provider のモデルは、`modelOverrides` の下の `reasoningEfforts: false` でレベルを失います。その状態で effort を選ぶと `UNSUPPORTED_REASONING_EFFORT` として拒否されます。DeepSeek 自身のルートにはこれらは不要です。そのモデルはすでに `off`、`low`、`high`、`max` を提供しており、`llm-deepseek.reasoningEffort` がピッカーの開始時の既定値を設定します:

```yaml
- id: llm-deepseek
  config:
    reasoningEffort: max
```

### リクエスト互換性

ゲートウェイは、到達可能なアドレスで有効なキーを保持していても、すべてのリクエストを拒否することがあります。pi-ai はリクエストの形（どのロールがシステムプロンプトを運ぶか、どのフィールドが出力を制限するか、thinking レベルがどう伝わるか）をエンドポイントの URL から決め、認識できないアドレスは OpenAI 自身であるかのように扱います。ほとんどの OpenAI 互換ゲートウェイは、OpenAI が受け付けるものの少なくとも1つを拒否します。

その大半は2つで説明できます。reasoning を宣言したモデルはシステムプロンプトが `role: "developer"` として送信され、多くのゲートウェイがこれを即座に拒否します。また出力上限は `max_completion_tokens` として送信され、`max_tokens` しか知らないサーバーはこれを拒否します。フォームにはどちらのフィールドもないため、`$DSH_HOME/profiles/<profile>/cordis.patch.yml` のルートで修正します:

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-gateway:
        apiKeyEnv: GATEWAY_API_KEY
        api: openai-completions
        baseURL: https://gateway.example/v1
        compat:
          supportsDeveloperRole: false
          maxTokensField: max_tokens
        models:
          - id: my-model
```

ルートの `compat` はそのモデルの既定値で、モデル自身の設定がフィールド単位で優先されるため、ルートを再記述せずに1つのモデルだけ修正できます:

```yaml
        models:
          - id: my-model
          - id: my-reasoner
            compat:
              thinkingFormat: deepseek
```

どちらにも設定されていないものは、そのモデルについてインストール済みカタログの値を保持し、カタログが記述していないものは pi-ai の検出に委ねられます。指定したすべてのスイッチに値を与えてください。空のままのキー（`supportsDeveloperRole:`）は無視されるのではなく拒否されます。空の値はカタログの知識を消しつつ、代わりの何も示さないからです。どのプロトコルも受け付けない名前も拒否され、メッセージには利用可能な名前が列挙されます。

各スイッチはそれを宣言するプロトコルに属するため、ある `api` で有効なスイッチが別の `api` では拒否されることがあります。メッセージにはそのプロトコルが提供するものが示されます。上記の `input` と同様、スイッチはエンドポイントについての主張を述べるだけでチェックは行いません。ゲートウェイが実際には必要としないスイッチを設定すると、単に異なるリクエストが送信されます。

すべてのスイッチ、受け付ける値、それを取るプロトコルは、[生成された `dsh-llm-pi-ai` 設定リファレンス](../../../../docs/config-catalog.md#deepseek-aidsh-llm-pi-ai) の `PiAiCompatProfile` に掲載されています。これはソースから導出されるため、アダプターが受け付ける内容に遅れることはありません。

## トラブルシューティング

- **`MISSING_CREDENTIAL`** — Models ページで provider キーを保存するか、参照されている環境変数を指定してください。
- **`UNKNOWN_MODEL`** — 設定済みのモデルを選択するか、カスタム provider に不足しているモデルを追加してください。
- **利用可能なモデルの取得が 401 を返す** — キーを確認してください。モデル検出は OpenAI 互換の `GET /models` エンドポイントを呼び出します。それを提供しないエンドポイントではモデルを手動で入力してください。
- **利用可能なモデルの取得が `data` 配列も `models` オブジェクトも報告しない** — エンドポイントの一覧は検出が読まない形式です。モデルを手動で入力してください。
- **キーと URL は正しいのにゲートウェイがすべてのリクエストを拒否する** — リクエストの形が OpenAI と異なります。まずルートに `compat.supportsDeveloperRole: false` と `compat.maxTokensField: max_tokens` を設定してください。
- **reasoning モデルだけが失敗する** — pi-ai はそれらのシステムプロンプトを `developer` ロールとして送信し、ゲートウェイが拒否します。`compat.supportsDeveloperRole: false` を設定してください。
- **手動で入力したモデルで Effort メニューが表示されない** — レベルが宣言されていません。カスタムモデル API provider では、モデルの **Model options** で **Reasoning levels** をオンにします。それ以外では、`cordis.patch.yml` のモデルに `reasoningEfforts` を追加します。
- **`off` を設定しても DeepSeek モデルが推論を止めない** — 空の `off` は reasoning フィールドを一切送信せず、既定で推論するエンドポイントは推論を続けます。モデルまたはルートに `compat.thinkingFormat: deepseek` を設定してください。
- **compat スイッチが値なしとして拒否される** — コロンの後に何も書かれていないキーです。値を与えるか、キーを削除してインストール済みカタログの値を保持してください。
- **画像が送信前に拒否される** — モデルが画像モダリティを宣言していません。カスタム provider のモデルには `input: [text, image]` を与えます。DeepSeek 自身のルートでは、設定済みカタログから画像対応エントリ（既定では `deepseek-flash`）を選択し、ゲートウェイがそのモデルを画像入力で提供することを確認してください。
- **provider が画像を含むリクエストを拒否する** — モデルがエンドポイントで実際には提供しない画像を宣言しています。それを許可したリスト（モデルの `input` またはルートの `defaultInput`）から `image` を削除し、新しい session を開始してください。添付画像は session ログに残るため、session がそこから離れるまで同じリクエストが繰り返されます。
