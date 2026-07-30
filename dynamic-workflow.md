> From: https://chatgpt.com/c/6a674cd2-a780-83ee-af45-eb87f7f9db14

# you asked

message time: 2026-07-27 21:19:30

Claude code dynamic workflowで使われているワークフローを記したjavascriptに使われている知見と読み方を学びたいです。よく調査して説明してください

---

# chatgpt response

First, I’ll identify the specific “Dynamic Workflow” implementation, then analyze its JavaScript from primary sources by syntax, DSL, execution model, design patterns, and safe modification.

---

# chatgpt response

The key distinction is JavaScript as a compact control-plane DSL while subagents perform the actual work. Focus first on async/await, functions, arrays, JSON Schema, branching, iteration, and hooks.

---

# chatgpt response

I’ll compare the official minimal example with runtime constraints and a community reconstruction of `/deep-research`, clearly separating confirmed specifications from inferred design lessons, including validation and stopping conditions.

---

# chatgpt response

# 結論

Claude Code Dynamic WorkflowsのJavaScriptは、普通のWebアプリやNode.jsプログラムとは少し違います。

本質は、

> **JavaScriptを、複数のAIエージェントを動かすための制御言語として使う**

ことです。

したがって、読むときにはJavaScript文法だけでなく、次の3層を分けて理解する必要があります。

1. **JavaScriptの層**
   `const`、配列、オブジェクト、関数、`await`、`map`、`filter`など。

2. **Workflow DSLの層**
   `agent()`、`parallel()`、`pipeline()`、`phase()`、`args`など。これらは標準JavaScriptではなく、Claude CodeのWorkflow Runtimeが提供します。

3. **エージェント設計の層**
   並列探索、依存関係、検証、反証、集約、停止条件、コスト制御など。

最も重要な読み方は、コードを単に上から下へ読むのではなく、

> **`agent()`をノード、変数をデータの流れ、`await`を依存関係として、処理グラフを復元する**

ことです。

Claude Codeの公式ドキュメントでも、Dynamic Workflowは「JavaScriptでサブエージェントをオーケストレーションする仕組み」とされ、通常の対話型サブエージェントと異なり、次の処理をモデルではなくスクリプトが決めます。中間結果は会話ではなくJavaScript変数に保持されます。現在の公式要件はClaude Code v2.1.154以降です。([Claude](https://code.claude.com/docs/en/workflows))

---

# 1. Dynamic Workflowの正しいメンタルモデル

全体像は次のようになります。

```text
ユーザーの依頼
    ↓
ClaudeがWorkflow JavaScriptを生成
    ↓
Workflow Runtimeがスクリプトを実行
    ↓
┌──────────────────────────────┐
│ JavaScriptによる制御         │
│                              │
│ agent()      調査担当         │
│ agent()      実装担当         │
│ agent()      検証担当         │
│ pipeline()   項目ごとの処理   │
│ parallel()   並列処理         │
│ if / loop    分岐・再試行     │
└──────────────────────────────┘
    ↓
中間結果をJavaScript変数に保存
    ↓
検証・絞り込み・集約
    ↓
return 最終結果
```

ここでは役割が明確に分かれます。

| 部分 | 責任 |
|---|---|
| JavaScript | 誰を、いつ、何個、どの順番で動かすか |
| `agent()`のプロンプト | 各エージェントが何を考えるか |
| JSON Schema | エージェント間で受け渡すデータ形式 |
| 検証エージェント | 内容が本当に正しいかを確認する |
| `return` | Workflow全体の成果物を確定する |

つまり、

- **JavaScriptは制御面**
- **サブエージェントは知的作業面**
- **Schemaは両者をつなぐ契約**

と考えると分かりやすくなります。

---

# 2. Subagent、Skill、Hook、Agent Teamとの違い

Dynamic Workflowを理解するには、近い概念との区別が重要です。

| 機能 | 主な役割 | 誰が次の処理を決めるか |
|---|---|---|
| Subagent | 特定作業を行う専門家 | 呼び出し元のClaude |
| Skill | 手順・知識・規約を与える | Claude |
| Hook | 特定イベントに反応する | 事前定義されたルール |
| Agent Team | リーダーと複数エージェントの協調 | 主にリーダーエージェント |
| Dynamic Workflow | 再現可能な複数エージェント処理 | JavaScript |

Subagentには独立したコンテキスト、システムプロンプト、ツール権限、モデル、最大ターン数などを設定できます。Subagentそのものが「作業者」であり、Dynamic Workflowはそれらを動かす「監督・工程表」です。([Claude](https://code.claude.com/docs/en/sub-agents))

Hookは、ツール実行前後、セッション開始時などのイベントに反応して、コマンドを拒否したり、フォーマットを実行したり、通知したりする仕組みです。Hookがイベント駆動の安全装置であるのに対し、Workflowはタスク全体の依存グラフを表します。([Claude](https://code.claude.com/docs/en/hooks-guide))

---

# 3. まず読めるようになるべき最小Workflow

以下は、Dynamic Workflowの典型構造を学ぶための例です。

```js
export const meta = {
  name: "review-auth-files",
  description: "認証関連ファイルを発見し、並列レビューして集約する",
};

const discovery = await agent(
  `
  src/ 以下から、認証処理を担当しているファイルを最大5個見つけてください。
  推測ではなく、実際にファイルを読んで判断してください。
  `,
  {
    label: "discover-auth-files",
    schema: {
      type: "object",
      required: ["files"],
      properties: {
        files: {
          type: "array",
          items: { type: "string" },
        },
      },
    },
  },
);

const reviews = await pipeline(
  discovery.files,
  (file) =>
    agent(
      `
      ${file} をセキュリティの観点からレビューしてください。

      条件:
      - 実際のコードを根拠にする
      - 推測だけの指摘は出さない
      - 問題がなければ findings を空配列にする
      `,
      {
        label: `review:${file}`,
        schema: {
          type: "object",
          required: ["findings"],
          properties: {
            findings: {
              type: "array",
              items: {
                type: "object",
                required: ["file", "summary", "evidence"],
                properties: {
                  file: { type: "string" },
                  summary: { type: "string" },
                  evidence: { type: "string" },
                },
              },
            },
          },
        },
      },
    ),
);

const findings = reviews
  .filter(Boolean)
  .flatMap((review) => review.findings);

return {
  reviewedFiles: discovery.files,
  findings,
};
```

公式の保存Workflowも、概ね次の構造になっています。

```text
export const meta
↓
agent() で探索
↓
探索結果の配列
↓
pipeline() で項目ごとにagent()
↓
filter / mapなどで整形
↓
return
```

公式ドキュメントでは、Workflow本体はトップレベル`await`を使えるJavaScriptで、`agent()`は1つのサブエージェントを起動し、`pipeline()`は配列の各項目について処理を実行するものとして説明されています。([Claude](https://code.claude.com/docs/en/workflows))

---

# 4. 上のコードを1行ずつ読む

## 4.1 `export const meta`

```js
export const meta = {
  name: "review-auth-files",
  description: "認証関連ファイルを発見し、並列レビューして集約する",
};
```

これはWorkflowのメタデータです。

- `const`：変数の再代入を禁止する
- `{ ... }`：オブジェクト
- `export`：Runtimeから参照できるように公開する
- `name`：Workflow名
- `description`：目的

ここで注意したいのは、通常のNode.jsモジュールに見えても、ファイル全体はClaude CodeのWorkflow Runtimeによって特別に解釈されることです。

特に後半のトップレベル`return`は、通常のJavaScriptモジュールにそのまま書く構造ではありません。Workflow Runtimeがスクリプト本体を実行可能な形に包んでいると考えるのが適切です。

---

## 4.2 `await agent(...)`

```js
const discovery = await agent(...);
```

読み方は、

> 「探索エージェントを起動し、その結果が返るまでこの依存箇所を待ち、結果を`discovery`に保存する」

です。

`await`はPromiseの完了を待ち、その結果を取り出します。トップレベル`await`はモジュール形式など特定の実行環境で利用でき、Dynamic Workflow Runtimeもこれを許可しています。([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await))

重要なのは、`await`を単に「待つ命令」と考えないことです。

Workflowでは、

> **`await`はデータ依存関係を表す**

と読むべきです。

```js
const discovery = await agent(...);

const reviews = await pipeline(discovery.files, ...);
```

この場合、`pipeline`は`discovery.files`を必要とするため、探索が終わるまで開始できません。

グラフにすると次の通りです。

```text
discover agent
     ↓ discovery.files
review pipeline
```

---

## 4.3 `schema`

```js
schema: {
  type: "object",
  required: ["files"],
  properties: {
    files: {
      type: "array",
      items: { type: "string" },
    },
  },
}
```

これはJSON Schemaです。

読み方は次の通りです。

```text
結果全体は object
└─ files は必須
   └─ files は array
      └─ 各要素は string
```

重要なポイントは、`properties`に書いただけでは必須にならないことです。必須項目は`required`にも指定します。配列の各要素の型は`items`で指定します。([JSON Schema](https://json-schema.org/understanding-json-schema/reference/object))

Schemaを使わない場合、エージェントは次のような自然文を返すかもしれません。

```text
認証関連と思われるファイルは以下です。
1. src/auth/login.ts
2. src/auth/session.ts
```

この文章から次のエージェント用の配列を取り出すのは不安定です。

Schemaを使うと、

```json
{
  "files": [
    "src/auth/login.ts",
    "src/auth/session.ts"
  ]
}
```

として扱えます。

ただし、Schemaが保証するのは主に**形式**です。

```text
Schema検証に通った
≠
内容が事実として正しい
```

そのため、本格的なWorkflowでは、構造化出力に加えて別の検証エージェントが必要です。公式のStructured Outputsも、JSONの欠損、型の揺れ、不正な構造を防ぐための仕組みとして説明されています。([Claude Platform](https://platform.claude.com/docs/en/build-with-claude/structured-outputs))

---

## 4.4 `discovery.files`

```js
discovery.files
```

これは、先ほどのオブジェクトから`files`プロパティを取り出しています。

```js
discovery = {
  files: [
    "src/auth/login.ts",
    "src/auth/session.ts",
  ],
};
```

なので、

```js
discovery.files
```

は、

```js
[
  "src/auth/login.ts",
  "src/auth/session.ts",
]
```

です。

Workflowを読むとき、配列は非常に重要です。

> **配列は、複数エージェントに配る仕事の一覧である**

ことが多いからです。

ファイル一覧、Issue一覧、仮説一覧、テスト失敗一覧、レビュー指摘一覧などが、そのままワークキューになります。

---

## 4.5 アロー関数

```js
(file) => agent(...)
```

これはアロー関数です。

通常の関数で書けば、おおよそ次と同じ意味です。

```js
function reviewFile(file) {
  return agent(...);
}
```

短いアロー関数では、波括弧を省略すると式の結果が自動的に返されます。

```js
(file) => agent(...)
```

は、

```js
(file) => {
  return agent(...);
}
```

と同じです。アロー関数は、配列処理やコールバックを簡潔に書くために多用されます。([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions))

Workflowでは、

```js
(file) => agent(...)
```

を、

> 「各fileに対して、次のagentを生成する規則」

と読みます。

---

## 4.6 テンプレートリテラル

```js
label: `review:${file}`
```

バッククォートで囲まれた文字列では、`${...}`を使って変数を埋め込めます。

```js
file = "src/auth/login.ts"
```

なら、

```js
`review:${file}`
```

は、

```text
review:src/auth/login.ts
```

になります。

プロンプトでも同じです。

```js
`
${file} をレビューしてください。
`
```

これは、同じ処理を複数ファイルに適用するためのテンプレートです。

---

## 4.7 `filter(Boolean)`

```js
reviews.filter(Boolean)
```

`filter()`は、条件が真になる要素だけを残した新しい配列を返します。([MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/filter))

`Boolean`を直接渡すと、次のようなfalsyな値が除去されます。

```js
false
null
undefined
0
""
NaN
```

したがって、

```js
reviews.filter(Boolean)
```

は概ね、

```js
reviews.filter((review) => Boolean(review))
```

という意味です。

便利ですが、堅牢なWorkflowでは注意が必要です。

例えば、

```text
null      = agent実行失敗
undefined = 結果欠損
false     = 検証不合格
[]        = 指摘なし
```

を区別したい場合、`filter(Boolean)`では情報を失います。

より堅牢にするなら、結果に明示的な状態を持たせます。

```json
{
  "status": "success",
  "findings": []
}
```

または、

```json
{
  "status": "failed",
  "error": "File could not be read"
}
```

とします。

---

## 4.8 `flatMap`

```js
.flatMap((review) => review.findings)
```

各レビュー結果が次の形だとします。

```js
[
  {
    findings: [
      { summary: "Issue A" },
      { summary: "Issue B" },
    ],
  },
  {
    findings: [
      { summary: "Issue C" },
    ],
  },
]
```

`map`だけなら、

```js
[
  [
    { summary: "Issue A" },
    { summary: "Issue B" },
  ],
  [
    { summary: "Issue C" },
  ],
]
```

という二重配列になります。

`flatMap`は、変換した後に1段階平坦化します。

```js
[
  { summary: "Issue A" },
  { summary: "Issue B" },
  { summary: "Issue C" },
]
```

これは典型的な**fan-in、つまり複数結果の集約**です。

---

## 4.9 `return`

```js
return {
  reviewedFiles: discovery.files,
  findings,
};
```

これがWorkflowの最終成果物です。

Workflowを読むときは、実は最初に`return`を見るのが有効です。

```text
最終的に何を返すのか
↓
その値を作るために何が必要か
↓
さらにその入力はどこから来るか
```

と、逆向きにたどることで依存関係が分かります。

---

# 5. `agent()`は普通の関数ではない

JavaScriptとして見ると、

```js
const result = await agent(prompt, options);
```

は単なる非同期関数呼び出しに見えます。

しかし概念的には、次の処理を行っています。

```text
新しいサブエージェントを生成
↓
独立したコンテキストを与える
↓
プロンプトを実行させる
↓
必要ならツールを使わせる
↓
Schemaに合う結果を返させる
↓
JavaScript値として受け取る
```

したがって、次の2つは別々に読む必要があります。

```js
agent(
  "何をさせるか",
  {
    label: "観測用の名前",
    schema: "戻り値の型",
  },
)
```

## プロンプト部分

これは「自然言語で書かれた関数仕様」です。

優れたプロンプトには、次の要素が入ります。

```text
役割:
  何の専門家として動くか

対象:
  どのファイル、Issue、仮説を扱うか

情報源:
  リポジトリ、テスト、公式文書など何を根拠にするか

除外条件:
  何を指摘してはいけないか

成功条件:
  何が確認できれば完了か

出力意味:
  各フィールドが何を意味するか
```

## Schema部分

これは「機械が理解できる関数の戻り値型」です。

TypeScript風に書けば、

```ts
type ReviewResult = {
  findings: Array<{
    file: string;
    summary: string;
    evidence: string;
  }>;
};
```

に近い役割です。

つまり`agent()`は、

```ts
async function review(file: string): Promise<ReviewResult>
```

に似たものとして読むと理解しやすくなります。

ただし、その関数本体が通常のアルゴリズムではなく、LLMによる推論です。

---

# 6. 逐次実行、並列実行、Pipelineの違い

## 6.1 逐次実行

```js
const analysis = await agent("問題を分析してください");

const verification = await agent(
  `次の分析を検証してください:\n${analysis}`,
);
```

グラフは次の通りです。

```text
analysis
   ↓
verification
```

`verification`は`analysis`を必要とするので、逐次実行で正しいです。

---

## 6.2 独立した仕事の並列実行

概念的には次のように書きます。

```js
const [security, performance, tests] = await parallel([
  () => agent("セキュリティ面を調査してください"),
  () => agent("性能面を調査してください"),
  () => agent("テスト不足を調査してください"),
]);
```

グラフは次の通りです。

```text
              ┌─ security
start ────────┼─ performance
              └─ tests
                    ↓
               全結果を待つ
```

ここで、

```js
() => agent(...)
```

と関数で包んでいる点が重要です。

これは、まだ実行せず、

> 「実行するとagentが始まる仕事」

を`parallel`に渡している、と読めます。

関数で包まなければ、`parallel`がスケジューリングする前に関数呼び出しが評価される可能性があります。

公式のSDK Referenceは`agent()`、`parallel()`、`pipeline()`、`phase()`をWorkflow向けのプリミティブとして列挙しています。`parallel`がthunk、つまり`() => agent(...)`形式を受け取る読み方は、Anthropic実装を参考にした公開互換実装でも明示されています。ただし、細かな引数や失敗時の挙動はバージョン依存になり得るため、最終的には現在の公式Referenceと生成されたraw scriptを正とするべきです。([Claude](https://code.claude.com/docs/en/agent-sdk/typescript))

---

## 6.3 Pipeline

```js
const reviews = await pipeline(
  files,
  (file) => agent(`Review ${file}`),
);
```

これは、

```text
filesの各項目についてagentを起動する
```

処理です。

```text
file A ─→ review A
file B ─→ review B
file C ─→ review C
```

公式例では、探索で得たファイル配列を`pipeline()`に渡し、各ファイルを個別のエージェントに監査させています。([Claude](https://code.claude.com/docs/en/workflows))

複数段階のPipelineは概念的には次のようになります。

```text
file A → review A → verify A
file B → review B → verify B
file C → review C → verify C
```

ここで大切なのは、

```text
review Aが終わればverify Aを始められる
review Bの終了まで待つ必要はない
```

という**項目ごとの依存関係**です。

公開互換実装では、各項目のステージは逐次、異なる項目同士は重ねて進める形として説明されています。これはデータ処理におけるストリーミングPipelineや、工場のベルトコンベアに近い構造です。([GitHub](https://github.com/michaelliv/pi-dynamic-workflows))

---

## 6.4 3つの違い

```text
逐次await
A → B → C
各処理が前の結果に依存

parallel
┌→ A ─┐
├→ B ─┼→ 全件完了
└→ C ─┘
互いに独立

pipeline
item1 → A1 → B1
item2 → A2 → B2
item3 → A3 → B3
同一項目内は逐次、項目間は並行可能
```

Workflowの性能を読むときは、`agent()`の数以上に、

> **どこに全体待ちのBarrierがあるか**

を見ることが重要です。

---

# 7. JavaScriptから復元すべき処理グラフ

先ほどの例は、次のグラフです。

```text
                     ┌→ review(file 1) ─┐
discover → files[] ──┼→ review(file 2) ─┼→ flatten → return
                     ├→ review(file 3) ─┤
                     └→ review(file 4) ─┘
```

ここに検証を追加すると、

```text
                     ┌→ review 1 → verify 1 ─┐
discover → files[] ──┼→ review 2 → verify 2 ─┼→ deduplicate
                     ├→ review 3 → verify 3 ─┤
                     └→ review 4 → verify 4 ─┘
                                                  ↓
                                               summarize
                                                  ↓
                                                return
```

コードを読むときは、自分でこの図を作ります。

次の記号変換が便利です。

| コード | グラフ上の意味 |
|---|---|
| `agent()` | ノード |
| 変数 | ノード間を流れるデータ |
| `await` | 依存関係 |
| 配列 | ワークキュー |
| `parallel()` | 並列分岐 |
| `pipeline()` | 項目別の処理列 |
| `if` | 条件分岐 |
| `for` / `while` | 反復 |
| `schema` | ノード間のデータ契約 |
| `return` | 最終出力 |

---

# 8. Workflowに使われている主要な設計知識

## 8.1 Fan-out / Fan-in

```text
1つの入力
↓
複数エージェントに分散
↓
結果を1つに集約
```

例：

```text
PR diff
├─ security reviewer
├─ test reviewer
├─ architecture reviewer
└─ API compatibility reviewer
       ↓
   final aggregator
```

Dynamic Workflowの最も基本的な形です。

---

## 8.2 Map-Reduce

配列の各項目を処理する部分がMapです。

```js
const reviews = await pipeline(
  files,
  (file) => agent(...),
);
```

複数の結果を集約する部分がReduceです。

```js
const allFindings = reviews.reduce(
  (acc, review) => [...acc, ...review.findings],
  [],
);
```

`map()`は各配列要素に関数を適用して新しい配列を作り、`reduce()`は複数要素を1つの累積値にまとめます。([MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/map))

---

## 8.3 Planner–Executor–Verifier

高品質なWorkflowで特に重要な構造です。

```text
Planner
  何を調べるか、作業単位を決める
      ↓
Executor
  各作業を実施する
      ↓
Verifier
  結果を独立に検証する
      ↓
Aggregator
  確認済み結果だけをまとめる
```

JavaScriptでは次のように現れます。

```js
const plan = await agent("調査項目を分解する");

const executions = await pipeline(
  plan.tasks,
  (task) => agent(`実行: ${task}`),
);

const verifications = await pipeline(
  executions,
  (execution) => agent(`独立に検証: ${JSON.stringify(execution)}`),
);
```

---

## 8.4 Adversarial Verification

単に「確認してください」ではなく、

```text
この主張が誤っている証拠を探す
反例を探す
再現しない条件を探す
別の解釈を検討する
```

という役割を与えます。

```js
const challenge = await agent(`
  次の指摘を支持するのではなく、誤りである可能性を調べてください。
  コード、テスト、公式仕様から反証できる場合は rejected にしてください。
`);
```

公式Workflow例にも、最初の結果を別エージェントで敵対的に検証するパターンや、候補を順位付け・重複排除するパターンがあります。([Claude](https://code.claude.com/docs/en/workflows))

---

## 8.5 Fixed-point / Saturation Loop

調査結果が増えなくなるまで続ける方式です。

```js
let known = [];
let noProgressRounds = 0;

for (let round = 0; round < 5; round += 1) {
  const result = await agent(...);

  const newItems = findNewItems(known, result.items);

  if (newItems.length === 0) {
    noProgressRounds += 1;
  } else {
    noProgressRounds = 0;
    known = [...known, ...newItems];
  }

  if (noProgressRounds >= 2) {
    break;
  }
}
```

これは数学的には、更新しても集合が変わらなくなる固定点に近い考え方です。

公式例にも、

- チェッカーが合格するまで修正する
- 新しい結果が出なくなるまで探索する
- 進展がない場合に停止する

といったループがあります。([Claude](https://code.claude.com/docs/en/workflows))

必ず必要なのは次の3つです。

```text
最大反復回数
成功条件
進展なしの停止条件
```

これらがなければ、エージェントが同じ指摘と修正を繰り返す可能性があります。

---

## 8.6 Contract-first Design

各エージェントを作る前に、出力Schemaを決めます。

悪い順序：

```text
とりあえずエージェントを動かす
↓
返ってきた文章を見て後から解析方法を考える
```

良い順序：

```text
次工程が必要とするデータを決める
↓
Schemaを決める
↓
そのSchemaを返すようにPromptを設計する
```

例えば検証工程が必要とするものが、

```ts
{
  verdict: "confirmed" | "rejected" | "uncertain";
  evidence: string[];
  reason: string;
}
```

なら、前段のレビュー結果も検証可能な形に揃えます。

---

## 8.7 Isolation

複数エージェントが同時に同じファイルを編集すると、競合します。

```text
Agent Aがfile.tsを変更
Agent Bも同じfile.tsを変更
↓
上書き、競合、テスト結果の混線
```

そのため、編集Workflowでは、

- Git worktree
- コピーされた作業ディレクトリ
- ファイル単位の所有権
- 1つの統合担当だけが最終反映

などを使います。

公式のWorkflow例にも、移行作業を隔離されたコピーで実行し、後から結果を比較・統合するパターンがあります。([Claude](https://code.claude.com/docs/en/workflows))

---

## 8.8 Observability

Workflowでは、内部で何が起きているか分からなくなりやすいため、

```js
label: `review:${file}`
```

や、

```js
phase(...)
```

を使って処理を識別します。

`label`には役割だけでなく、対象を含めると追跡しやすくなります。

悪い例：

```js
label: "review"
```

良い例：

```js
label: `security-review:${file}`
```

さらに、`phase()`や`meta.phases`は進捗表示上の処理グループを表す用途があります。これは依存関係そのものというより、Workflowの観測可能性を高める仕組みとして読むべきです。([Claude](https://code.claude.com/docs/en/agent-sdk/typescript))

---

# 9. Workflowを読むための7段階

## 第1段階：最終`return`を見る

```js
return {
  findings,
  summary,
};
```

最終成果物を把握します。

確認することは、

```text
何を返すWorkflowなのか
ファイル変更なのか
調査結果なのか
検証済み情報なのか
```

です。

---

## 第2段階：入力を見る

主な入力源は、

```js
args
```

または、先行エージェントの結果です。

```js
const target = args?.target ?? "src/";
```

`args`はWorkflow起動時に渡される構造化入力で、引数がなければ`undefined`になります。([Claude](https://code.claude.com/docs/en/workflows))

読みながら、

```text
外部入力
固定値
エージェント生成値
```

を区別します。

---

## 第3段階：すべての`agent()`を列挙する

次の表を作ると読みやすくなります。

| Agent | 入力 | 役割 | 出力 | 副作用 |
|---|---|---|---|---|
| discover | target | ファイル探索 | files[] | なし |
| review | file | コードレビュー | findings[] | なし |
| verify | finding | 真偽確認 | verdict | なし |
| fix | confirmed finding | 修正 | patch result | あり |
| test | repository state | テスト | test result | あり得る |

特に、ファイルを変更する可能性があるエージェントを明示します。

---

## 第4段階：変数を矢印として結ぶ

```js
const files = ...
const reviews = pipeline(files, ...)
const verified = pipeline(reviews, ...)
return verified
```

なら、

```text
files → reviews → verified → return
```

です。

コード量が多くても、データフローにすると小さなグラフになります。

---

## 第5段階：並列性を確認する

次を色分けします。

```text
逐次:
  await agent()

並列:
  parallel([...])

項目単位:
  pipeline(items, ...)

通常の同期処理:
  map / filter / reduce / sort
```

特に、不要な逐次化を探します。

```js
const security = await agent(...);
const performance = await agent(...);
```

両者が独立なら、並列化候補です。

一方で、

```js
const verification = await agent(
  `Verify ${security}`,
);
```

は依存しているため逐次で正しいです。

---

## 第6段階：正しさの根拠を見る

次を確認します。

```text
Schemaがあるか
証拠フィールドがあるか
独立検証があるか
反証工程があるか
テストがあるか
重複排除があるか
不確実性を表現できるか
```

次のSchemaは不十分です。

```js
{
  type: "object",
  properties: {
    correct: { type: "boolean" },
  },
}
```

なぜなら、根拠なしで`true`を返せるからです。

よりよい構造は、

```js
{
  type: "object",
  required: ["verdict", "evidence", "reason"],
  properties: {
    verdict: {
      type: "string",
      enum: ["confirmed", "rejected", "uncertain"],
    },
    evidence: {
      type: "array",
      items: { type: "string" },
    },
    reason: {
      type: "string",
    },
  },
}
```

です。

---

## 第7段階：失敗・停止・コストを見る

確認項目は次の通りです。

```text
Agentが失敗したらどうするか
一部だけ成功した場合どうするか
ループの最大回数はあるか
同じ作業を繰り返さないか
何個のAgentが生成されるか
大量の出力を最終Agentに渡していないか
編集競合は起きないか
```

---

# 10. JavaScriptとして最低限学ぶべき範囲

Dynamic Workflowを読む目的なら、JavaScript全体を先に学ぶ必要はありません。

優先順位は次の通りです。

## 最優先

```js
const
```

```js
{ key: value }
```

```js
[item1, item2]
```

```js
array.property
```

```js
(item) => expression
```

```js
await promise
```

```js
`text ${value}`
```

```js
if / else
```

```js
for / while / break
```

---

## 配列処理

```js
array.map(...)
array.filter(...)
array.flatMap(...)
array.reduce(...)
array.find(...)
array.some(...)
array.every(...)
array.sort(...)
```

Workflowでは、配列がワークキューになるため、通常のWeb開発以上に配列処理が重要です。

---

## 分割代入

```js
const [security, performance] = results;
```

```js
const { findings, summary } = review;
```

これは配列やオブジェクトから値を取り出す記法です。

---

## 安全な参照

```js
args?.target
```

`args`が`undefined`でもエラーにせず、`undefined`を返します。

```js
args?.target ?? "src/"
```

`args?.target`が`null`または`undefined`なら、`"src/"`を使います。

---

## 例外処理

```js
try {
  const result = await agent(...);
  return result;
} catch (error) {
  return {
    status: "failed",
    message: String(error),
  };
}
```

ただし、Workflow Runtime独自の失敗処理と競合する場合があるため、すべてを雑に`catch`して正常扱いにしないことが重要です。

---

# 11. Runtime上の重要な制約

Dynamic WorkflowのJavaScriptは隔離されたRuntimeで実行されます。

重要な制約は次の通りです。

- JavaScript本体から直接ファイルシステムやシェルを操作するのではなく、I/Oはサブエージェントに行わせる
- 中間結果はJavaScript変数に保持される
- 実行中の追加ユーザー入力は、原則として権限確認以外受け取らない
- 同時実行は最大16エージェント
- 1 Workflowあたり最大1000エージェント
- 実行情報はセッション内で保持され、同一セッションなら完了済みエージェント結果を再利用して再開できる
- セッションをまたいだ永続的再開ではない

これらはWorkflow設計に直接影響します。途中で人間の判断を入れたい場合は、巨大な1本のWorkflowにせず、「調査Workflow」と「承認後の修正Workflow」に分けるべきです。([Claude](https://code.claude.com/docs/en/workflows))

現在のサイズ設定では、おおよそ、

```text
small         5未満
medium        15未満
large         50未満
unrestricted  それ以上も許可
```

という区分があり、現在のデフォルトは`medium`です。学習時は`small`にして、最大3〜4エージェント程度から始める方が構造を追いやすくなります。([Claude](https://code.claude.com/docs/en/workflows))

また、25エージェント超または推定150万トークン超では警告が表示されるため、Agent数だけでなく、各Agentが読むコンテキストと返す出力量も設計対象です。([Claude](https://code.claude.com/docs/en/workflows))

---

# 12. よくある悪いWorkflow

## 12.1 すべてを1つのAgentにやらせる

```js
const result = await agent(`
  リポジトリ全体を調査し、
  問題を探し、
  修正し、
  テストし、
  レポートしてください。
`);
```

何を根拠にどこで失敗したか分かりません。

改善：

```text
discover
↓
review
↓
verify
↓
fix
↓
test
↓
summarize
```

と分割します。

---

## 12.2 Schemaが巨大すぎる

最初から何十項目も持つSchemaを作ると、

- 欠損
- 無意味な埋め草
- 不必要な出力
- トークン増加

が起きます。

各工程が本当に必要とする最小情報にします。

---

## 12.3 同じ情報を全Agentに渡す

```js
const hugeRepositorySummary = "...";

await parallel([
  () => agent(hugeRepositorySummary + taskA),
  () => agent(hugeRepositorySummary + taskB),
  () => agent(hugeRepositorySummary + taskC),
]);
```

では、同じ巨大コンテキストを何度も消費します。

改善策は、

- 対象ファイルを絞る
- Agentごとに必要な情報だけ渡す
- 共通知識をSkillやCLAUDE.mdに整理する
- 探索Agentに必要箇所を特定させる

ことです。

---

## 12.4 検証Agentが元Agentを追認するだけ

```js
agent("この結果が正しいことを確認してください")
```

では、元の結論に引っ張られます。

改善：

```text
この主張を反証する証拠を優先して探す
元の結論を前提にしない
コードとテストを独立に確認する
根拠がなければuncertainにする
```

と指示します。

---

## 12.5 無制限ループ

```js
while (!passed) {
  ...
}
```

これは危険です。

最低でも、

```js
for (let attempt = 0; attempt < 3; attempt += 1) {
  ...
}
```

のように上限を持たせます。

---

## 12.6 読み取りAgentと編集Agentが混ざっている

レビュー担当が勝手に編集すると、後段の検証対象が変化します。

```text
探索・レビュー・検証
  原則読み取り専用

修正
  明示的に編集許可

テスト
  修正後の状態を検証
```

と工程を分離します。

---

## 12.7 `filter(Boolean)`で失敗を消す

```js
const successful = results.filter(Boolean);
```

だけでは、

```text
問題なし
失敗
結果欠損
意図的なスキップ
```

を区別できません。

`status`をSchemaに入れる方が安全です。

---

# 13. 実際のWorkflowを読むためのチェックシート

JavaScriptを開いたら、次の順番で書き込みます。

```text
1. 最終returnは何か
2. argsなどの入力は何か
3. agent()はいくつあるか
4. 各agentの役割は何か
5. 各agentのSchemaは何か
6. 配列はどこで生成されるか
7. どこでfan-outするか
8. どこでfan-inするか
9. どのawaitが本当の依存関係か
10. 並列化できるのに逐次になっていないか
11. 検証者は実行者から独立しているか
12. 反証工程があるか
13. ループの成功条件と停止条件は何か
14. ファイルを書き換えるagentはどれか
15. 同時編集競合は起きないか
16. 一部失敗時の扱いは何か
17. Agent数とトークン量はどこで増えるか
18. 最終集約で情報を落としていないか
```

---

# 14. Claude Code上で学習する実践手順

現在は自然言語でDynamic Workflowを依頼でき、`ultracode`または`/effort ultracode`で明示的に起動できます。以前のバージョンでは`workflow`という語がトリガーでしたが、v2.1.160以降は`ultracode`に変更されています。([Claude](https://code.claude.com/docs/en/workflows))

最初の練習には、次の依頼が適しています。

```text
ultracode:

src/auth 以下を対象に、最大3ファイルだけを読み取り専用で監査する
小さなDynamic Workflowを作成してください。

構成:
1. 関連ファイルを発見するagent
2. 各ファイルを個別にレビューするpipeline
3. 指摘を独立に反証・検証するagent
4. confirmedだけを返す集約処理

要件:
- 実行前にraw JavaScriptを表示する
- 各agentに対象が分かるlabelを付ける
- すべての出力にJSON Schemaを付ける
- 最大3ファイル
- ファイルを編集しない
- ループを使用しない
```

生成された画面ではraw scriptを表示でき、`Ctrl+G`で編集してから実行できます。サブエージェントは親セッションのツール許可リストを継承し、編集モード上の権限動作にも注意が必要です。([Claude](https://code.claude.com/docs/en/workflows))

次の順番で段階的に高度化すると理解しやすくなります。

```text
第1段階
agent() 1個
入力 → 出力

第2段階
discover → pipeline
配列とfan-out

第3段階
JSON Schema
構造化データ契約

第4段階
review → verifier
実行と検証の分離

第5段階
複数観点のparallel
並列性

第6段階
重複排除とranking
fan-inと集約

第7段階
最大回数付き修正ループ
停止条件

第8段階
worktree上で編集とテスト
副作用と隔離
```

保存する場合は、プロジェクト単位なら`.claude/workflows/`、ユーザー単位なら`~/.claude/workflows/`が使われます。([Claude](https://code.claude.com/docs/en/workflows))

---

# 15. 最終的に身につけるべき読み替え

Dynamic WorkflowのJavaScriptを見たら、次のように頭の中で翻訳してください。

```text
const
→ 中間状態に名前を付ける

agent()
→ 独立したAI作業ノードを作る

await
→ この結果が次の処理に必要

配列
→ 複数の仕事の一覧

pipeline()
→ 各項目を処理ラインに流す

parallel()
→ 独立した仕事を同時に開始する

schema
→ エージェント間の型付き契約

if
→ 実行結果による分岐

for / while
→ 再試行または反復探索

break
→ 停止条件

filter
→ 不要な候補を除く

map
→ 各結果を変換する

reduce / flatMap
→ 複数結果を集約する

verifier agent
→ LLM出力の内容を独立検証する

phase / label
→ 実行を追跡可能にする

return
→ Workflowの正式な成果物
```

最も重要な原則を一文にまとめると、

> **Dynamic WorkflowのJavaScriptは、AIに何を考えさせるコードではなく、「どのAIに、何を渡し、どの順番で実行し、何を証拠として採用するか」を決めるコードです。**

そのため、上手に読むには、JavaScript文法よりも一段上の、

```text
データフロー
依存グラフ
並列性
出力契約
検証構造
停止条件
副作用
コスト
```

を復元することが核心になります。

---
Powered by [AI Exporter](https://saveai.net)