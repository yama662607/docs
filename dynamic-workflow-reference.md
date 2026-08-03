# Claude Code Dynamic Workflow 実践リファレンス

> **出典について**
> このドキュメントは、Claude Code (desktop, v2.1.219) の実行時に Claude 自身へ与えられている
> `Workflow` ツールの仕様記述を一次情報として書き起こしたものです。推測ではありません。
> 実体の所在は [§14 実体はどこにあるか](#14-実体はどこにあるか) を参照。
> 作成日: 2026-08-03

---

## 目次

1. [これは何か](#1-これは何か)
2. [呼び出し条件（オプトイン規則）](#2-呼び出し条件オプトイン規則)
3. [スクリプトの解剖](#3-スクリプトの解剖)
4. [`meta` ブロック](#4-meta-ブロック)
5. [スクリプト本体のフック一覧](#5-スクリプト本体のフック一覧)
6. [`agent()` の全オプション](#6-agent-の全オプション)
7. [構造化出力（schema）](#7-構造化出力schema)
8. [pipeline と parallel — バリア原理](#8-pipeline-と-parallel--バリア原理)
9. [公式コード例（全掲載）](#9-公式コード例全掲載)
10. [品質パターン](#10-品質パターン)
11. [ランタイム制約](#11-ランタイム制約)
12. [Resume（再開）の仕組み](#12-resume再開の仕組み)
13. [規模の決め方と Ultracode](#13-規模の決め方と-ultracode)
14. [実体はどこにあるか](#14-実体はどこにあるか)
15. [保存場所と実行時アーティファクト](#15-保存場所と実行時アーティファクト)
16. [ワークフローを読む・レビューするチェックリスト](#16-ワークフローを読むレビューするチェックリスト)
17. [アンチパターン](#17-アンチパターン)
18. [実戦例：本ドキュメント執筆セッションで実際に走らせたワークフロー](#18-実戦例本ドキュメント執筆セッションで実際に走らせたワークフロー)
19. [周辺機構との関係](#19-周辺機構との関係)
20. [確認済み事項と未確認事項の区別](#20-確認済み事項と未確認事項の区別)

---

## 1. これは何か

Dynamic Workflow は、**複数のサブエージェントを決定論的にオーケストレーションする JavaScript スクリプト**を実行する仕組みです。

仕様の定義文（原文）:

> Execute a workflow script that orchestrates multiple subagents deterministically.

重要なのは「決定論的に」という部分です。通常のサブエージェント呼び出しでは「次に何をするか」をモデルが判断しますが、Workflow では**制御フロー（ループ・分岐・fan-out）をスクリプトが決めます**。モデルは各ノードの中身だけを担当します。

### 何のために使うのか

仕様には3つの動機が明記されています。

| 動機 | 原文 | 意味 |
|---|---|---|
| 網羅性 | to be comprehensive (decompose and cover in parallel) | 分解して並列に潰す |
| 確信度 | to be confident (independent perspectives and adversarial checks before committing) | 独立した視点と敵対的検証を経てから確定する |
| スケール | to take on scale one context can't hold (migrations, audits, broad sweeps) | 単一コンテキストに載らない規模を扱う |

> The script is where you encode that structure: what fans out, what verifies, what synthesizes.
> （スクリプトとは、何を fan-out し、何を検証し、何を統合するかという**構造**を書く場所である）

### 実行モデル

- **バックグラウンド実行**。ツール呼び出しは即座に task ID を返し、完了時に `<task-notification>` が届く。
- 進行状況は `/workflows` でライブ表示できる。
- スクリプトは `script` パラメータに**インラインで渡す**。事前にファイルへ Write してはいけない（仕様に明記: "Pass the script inline via `script` — do not Write it to a file first"）。
- 起動のたびにスクリプトはセッションディレクトリへ自動保存され、そのパスが結果に含まれる。反復するときは**そのファイルを編集して `{scriptPath}` で再実行**する（全文を再送しない）。

### 3層で読む

```
┌─ 層1: JavaScript ────────────────────────────────┐
│  const / 配列 / オブジェクト / await / map / filter │
├─ 層2: Workflow DSL ──────────────────────────────┤
│  agent() parallel() pipeline() phase() log()      │
│  args budget workflow()                           │
├─ 層3: エージェント設計 ────────────────────────────┤
│  fan-out/fan-in / 検証構造 / 停止条件 / コスト制御   │
└──────────────────────────────────────────────────┘
```

読むときの核心は、**`agent()` をノード、変数をデータの流れ、`await` を依存関係として処理グラフを復元する**ことです。

---

## 2. 呼び出し条件（オプトイン規則）

Workflow は**ユーザーが明示的にオプトインしたときだけ**呼べます。仕様は強い調子で書かれています。

> ONLY call this tool when the user has explicitly opted into multi-agent orchestration.
> Workflows can spawn dozens of agents and consume a large amount of tokens; the user must
> request that scale, not have it inferred.

### オプトインとみなされる5条件

1. ユーザーのプロンプトに **`ultracode`** というキーワードが含まれている（system-reminder で確認できる）
2. セッションで **Ultracode が有効**になっている（system-reminder で確認できる）
3. ユーザーが**自分の言葉で**ワークフロー／マルチエージェント実行を求めた
   - "use a workflow" / "run a workflow" / "fan out agents" / "orchestrate this with subagents"
   - **重要**: 「ワークフローが有効そうなタスク」というだけでは条件を満たさない。ユーザーの言葉である必要がある
4. ユーザーが起動したスキル／スラッシュコマンドの指示に Workflow を呼べと書いてある
5. ユーザーが特定の名前付き／保存済みワークフローの実行を求めた

### それ以外の場合

> For any other task — even one that would clearly benefit from parallelism — do NOT call this tool.

代わりに、個別のサブエージェントには `Agent` ツールを使うか、**「マルチエージェントワークフローなら何ができて、だいたいどのくらいコストがかかるか」を簡潔に説明して、実行するか尋ねる**。その際「今後は "use a workflow" と言えばこの確認を飛ばせます」と伝えるとよい、とされています。

### ハイブリッドが正解であることが多い

仕様が明示的に推奨している進め方:

> The right move is often **hybrid**: scout inline first (list the files, find the channels, scope the diff)
> to discover the work-list, then call Workflow to pipeline over it.
> You don't need to know the shape before the *task* — only before the *orchestration step*.

つまり、**まず自分で偵察して作業リストを作り、そのリストに対して Workflow を回す**。タスク開始時点で全体像が分かっている必要はなく、オーケストレーション開始時点で分かっていればよい。

---

## 3. スクリプトの解剖

```js
export const meta = { /* 必須。純粋リテラル */ }

// ここから下がスクリプト本体（トップレベル await 可）
phase('Scan')
const items = await agent('...', { schema: SCHEMA })
const results = await pipeline(items, stage1, stage2)
return { results }
```

構造上のルール:

- ファイルは **必ず `export const meta = {...}` で始まる**
- 本体は async コンテキストで実行される → `await` を直接書ける
- トップレベル `return` がワークフローの戻り値になる
- **プレーンな JavaScript**。TypeScript ではない（型注釈 `: string[]`、interface、generics はパースエラー）

---

## 4. `meta` ブロック

```js
export const meta = {
  name: 'find-flaky-tests',
  description: 'Find flaky tests and propose fixes',   // one-line, shown in permission dialog
  phases: [                                            // one entry per phase() call
    { title: 'Scan', detail: 'grep test logs for retries' },
    { title: 'Fix', detail: 'one agent per flaky test' },
  ],
}
// script body starts here — use agent()/parallel()/pipeline()/phase()/log()
phase('Scan')
const flaky = await agent('grep CI logs for retry markers', {schema: FLAKY_SCHEMA})
...
```

### フィールド

| フィールド | 必須 | 内容 |
|---|---|---|
| `name` | ✅ | ワークフロー名 |
| `description` | ✅ | 一行説明。**権限ダイアログに表示される** |
| `whenToUse` | — | ワークフロー一覧に表示される |
| `phases` | — | `phase()` 呼び出し1つにつき1エントリ。`{title, detail}` |
| `model` | — | そのフェーズが特定モデルを使う場合に phase エントリへ付ける |

### 絶対規則：`meta` は純粋リテラル

> The `meta` object must be a PURE LITERAL — no variables, function calls, spreads, or template interpolation.

```js
// ✅ OK
export const meta = { name: 'audit', description: 'Audit the repo', phases: [{title: 'Scan'}] }

// ❌ NG — 変数
const N = 'audit'
export const meta = { name: N, ... }

// ❌ NG — テンプレート補間
export const meta = { name: `audit-${target}`, ... }

// ❌ NG — スプレッド
export const meta = { ...base, name: 'audit' }
```

### phase タイトルは完全一致でマッチする

`meta.phases` の `title` と `phase()` に渡す文字列は**厳密に一致**させます。一致しない `phase()` 呼び出しは、独自の進捗グループとして扱われます（エラーにはならない）。

---

## 5. スクリプト本体のフック一覧

### `agent(prompt, opts?) → Promise<any>`

サブエージェントを1体起動する。

- `schema` なし → 最終テキストを**文字列**で返す
- `schema` あり → 検証済みの**オブジェクト**を返す
- **`null` を返す場合がある**: ユーザーが実行中にそのエージェントをスキップした、またはリトライ後も終端的な API エラーで死亡した
  → `.filter(Boolean)` で除去する

サブエージェントには「**あなたの最終テキストが戻り値であり、人間へのメッセージではない**」と伝えられているため、生データを返してきます。

### `pipeline(items, stage1, stage2, ...) → Promise<any[]>`

各項目を全ステージに独立して通す。**ステージ間にバリアが無い**。

- 項目Aがステージ3にいる間、項目Bはステージ1にいてよい
- 実時間 = 最も遅い**単一項目のチェーン**。ステージごとの最遅値の総和ではない
- **多段処理のデフォルトはこちら**
- 各ステージのコールバックは **`(prevResult, originalItem, index)`** を受け取る
  → 後段で元の項目やインデックスを使えるので、ステージ1の戻り値に情報を詰め込む必要はない
- ステージが throw すると、その項目は `null` になり残りのステージはスキップされる

### `parallel(thunks) → Promise<any[]>`

タスクを並行実行する。**バリアである**（全 thunk を待つ）。

- thunk が throw した（またはエージェントがエラーになった）場合、その要素は `null` になる
- **呼び出し自体は決して reject しない** → 使う前に `.filter(Boolean)` が必要
- 全結果を本当に同時に必要とするときだけ使う

`() => agent(...)` と関数で包む理由: まだ実行せず「実行すると agent が始まる仕事」を渡すため。包まないと `parallel` がスケジューリングする前に評価されてしまう。

### `log(message) → void`

進捗メッセージを出力する。進捗ツリーの上にナレーター行として表示される。

### `phase(title) → void`

新しいフェーズを開始する。以降の `agent()` はこのタイトルでグループ化される。

> ⚠️ `pipeline()`/`parallel()` のステージ内では、グローバルな `phase()` 状態のレースを避けるため
> **`opts.phase` を明示的に使う**こと。同じ phase 文字列 → 同じグループボックス。

### `args`

Workflow ツールの `args` 入力がそのまま渡る（未指定なら `undefined`）。

> ⚠️ **配列・オブジェクトは実際の JSON 値として渡すこと**。JSON 文字列にしてはいけない。
> `args: ["a.ts", "b.ts"]` は OK。`args: "[\"a.ts\", ...]"` は1個の文字列として届き、`args.filter` / `args.map` が throw する。

### `budget`

ユーザーの「+500k」形式の指定によるトークン目標。

```js
budget.total       // number | null   目標値。指定がなければ null
budget.spent()     // number          このターンの出力トークン消費（メインループ＋全ワークフロー合算）
budget.remaining() // number          max(0, total - spent())。目標未設定なら Infinity
```

**目標はアドバイザリではなくハード上限**です。`spent()` が `total` に達すると以降の `agent()` は throw します。

プールは共有であり、ワークフローごとではありません。

### `workflow(nameOrRef, args?) → Promise<any>`

別のワークフローをインラインの子ステップとして実行し、その戻り値を返す。

- 名前を渡すと保存済みワークフローを起動、`{scriptPath}` を渡すと自前のスクリプトファイルを実行
- 子は**この実行の並行度上限・エージェントカウンタ・abort シグナル・トークン予算を共有**する
- 子のエージェントは `/workflows` 上で `▸ name` グループに表示され、トークンは `budget.spent()` に加算される
- `args` は子の `args` グローバルになる
- **ネストは1段のみ**。子の中で `workflow()` を呼ぶと throw
- 未知の名前 / 読めない scriptPath / 子の構文エラーで throw する → 必要なら catch

---

## 6. `agent()` の全オプション

```js
agent(prompt, {
  label:     string,     // 表示ラベルの上書き
  phase:     string,     // 進捗グループの明示指定（pipeline/parallel 内では必須級）
  schema:    object,     // JSON Schema。構造化出力を強制する
  model:     string,     // モデル上書き
  effort:    string,     // 'low' | 'medium' | 'high' | 'xhigh' | 'max'
  isolation: 'worktree', // 専用 git worktree で実行
  agentType: string,     // カスタムサブエージェント型
})
```

### `model`

> Default to omitting it — the agent inherits the main-loop model (the resolved session model),
> which is almost always correct. Only set it when you're highly confident a different tier fits
> the task; when unsure, omit.

**原則は省略**。セッションのモデルを継承するのがほぼ常に正しい。別のティアが確実に適していると分かるときだけ指定する。

### `effort`

省略するとセッションの effort を継承。安価な機械的ステージには `'low'`、最も難しい検証／判定ステージにだけ上位ティアを使う。

### `isolation: 'worktree'`

```
⚠️ EXPENSIVE — エージェント1体あたり ~200-500ms のセットアップ + ディスク消費
```

**エージェントが並列にファイルを書き換えて競合する場合にのみ**使う。変更が無ければ worktree は自動削除される。

### `agentType`

`Agent` ツールと同じレジストリから解決される（`'general-purpose'`, `'code-reviewer'` など）。デフォルトのワークフロー用サブエージェントの代わりに使う。

`schema` と**併用可能**（カスタムエージェントのシステムプロンプトに StructuredOutput 指示が追記される）。

---

## 7. 構造化出力（schema）

`schema` に JSON Schema を渡すと、サブエージェントは **StructuredOutput ツールの呼び出しを強制**され、`agent()` は検証済みオブジェクトを返します。

> validation happens at the tool-call layer so the model retries on mismatch

**パース不要**。不一致ならモデル側がリトライします。

### schema が保証するのは「形式」だけ

```
Schema検証に通った  ≠  内容が事実として正しい
```

したがって本格的なワークフローでは、構造化出力に**加えて**独立した検証エージェントが必要です。

### 弱い schema と強い schema

```js
// ❌ 弱い — 根拠なしに true を返せてしまう
{
  type: 'object',
  properties: { correct: { type: 'boolean' } },
}

// ✅ 強い — 判定・証拠・理由を分離し、不確実性を表現できる
{
  type: 'object',
  required: ['verdict', 'evidence', 'reason'],
  properties: {
    verdict:  { type: 'string', enum: ['confirmed', 'rejected', 'uncertain'] },
    evidence: { type: 'array', items: { type: 'string' } },
    reason:   { type: 'string' },
  },
}
```

### Contract-first で設計する

```
悪い順序: とりあえずエージェントを動かす → 返ってきた文章を見て後から解析方法を考える
良い順序: 次工程が必要とするデータを決める → Schema を決める → その Schema を返す Prompt を書く
```

### schema 設計の実務則

- **各工程が本当に必要とする最小情報**にする。何十項目もある schema は、欠損・埋め草・トークン増加を招く
- `status` フィールドを持たせると、`filter(Boolean)` で失われる情報（失敗／結果欠損／意図的スキップ／指摘なし）を区別できる
- 配列要素の型は `items`、必須は `required`（`properties` に書いただけでは必須にならない）

---

## 8. pipeline と parallel — バリア原理

これがワークフロー設計で**最も重要な判断**です。仕様は繰り返し強調しています。

> **DEFAULT TO pipeline().** Only reach for a barrier (parallel between stages) when you
> genuinely need ALL prior-stage results together.

### 3つの実行形

```
逐次 await
   A → B → C
   各処理が前の結果に依存

parallel（バリアあり）
   ┌→ A ─┐
   ├→ B ─┼→ 全件完了を待つ
   └→ C ─┘

pipeline（バリアなし）
   item1 → A1 → B1
   item2 → A2 → B2      ← item1 が B1 にいる間に item2 は A2 でよい
   item3 → A3 → B3
```

### バリアが正しい場合（これだけ）

ステージ N が、ステージ N-1 の**全結果にまたがる文脈**を必要とするときだけ:

- **重複排除／マージ** — 高コストな後段処理の前に、全結果セットに対して行う必要がある
- **早期終了** — 総件数がゼロなら検証を丸ごとスキップする
- ステージ N のプロンプトが「**他の findings**」を比較対象として参照する

### バリアの理由にならないもの

| 言い訳 | 反論 |
|---|---|
| 「先に flatten / map / filter したい」 | pipeline のステージ内でやれ: `pipeline(items, stageA, r => transform([r]).flat(), stageB)` |
| 「ステージは概念的に別物だ」 | それは pipeline がモデル化しているもの。**別ステージ ≠ 同期されたステージ** |
| 「コードがきれいになる」 | バリアのレイテンシは実在する。5つの finder のうち最遅が最速の3倍なら、速い finder のアイドル時間の 2/3 を捨てている |

### スメルテスト（原文の判定法）

```js
const a = await parallel(...)
const b = transform(a)        // flatten, map, filter — no cross-item dependency
const c = await parallel(b.map(...))
```

> that middle transform doesn't need the barrier. Rewrite as a pipeline with the transform inside a stage.
> **When in doubt: pipeline.**

### 性能を読むときの着眼点

> `agent()` の数以上に、**どこに全体待ちのバリアがあるか**を見る。

---

## 9. 公式コード例（全掲載）

仕様に含まれるコード例をすべて掲載します。

### 9.1 正準の多段パターン（pipeline デフォルト、各次元が終わり次第検証へ）

```js
export const meta = {
  name: 'review-changes',
  description: 'Review changed files across dimensions, verify each finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
const DIMENSIONS = [{key: 'bugs', prompt: '...'}, {key: 'perf', prompt: '...'}]
const results = await pipeline(
  DIMENSIONS,
  d => agent(d.prompt, {label: `review:${d.key}`, phase: 'Review', schema: FINDINGS_SCHEMA}),
  review => parallel(review.findings.map(f => () =>
    agent(`Adversarially verify: ${f.title}`, {label: `verify:${f.file}`, phase: 'Verify', schema: VERDICT_SCHEMA})
      .then(v => ({...f, verdict: v}))
  ))
)
const confirmed = results.flat().filter(Boolean).filter(f => f.verdict?.isReal)
return { confirmed }
// Dimension 'bugs' findings verify while dimension 'perf' is still reviewing. No wasted wall-clock.
```

### 9.2 バリアが**正しい**場合 — 全 findings を重複排除してから高コストな検証へ

```js
const all = await parallel(DIMENSIONS.map(d => () => agent(d.prompt, {schema: FINDINGS_SCHEMA})))
const deduped = dedupeByFileAndLine(all.filter(Boolean).flatMap(r => r.findings))  // <-- genuinely needs ALL at once
const verified = await parallel(deduped.map(f => () => agent(verifyPrompt(f), {schema: VERDICT_SCHEMA})))
```

### 9.3 loop-until-count — 目標件数まで積み上げる

```js
const bugs = []
while (bugs.length < 10) {
  const result = await agent("Find bugs in this codebase.", {schema: BUGS_SCHEMA})
  bugs.push(...result.bugs)
  log(`${bugs.length}/10 found`)
}
```

### 9.4 loop-until-budget — ユーザーの「+500k」指定に深さを合わせる

```js
const bugs = []
while (budget.total && budget.remaining() > 50_000) {
  const result = await agent("Find bugs in this codebase.", {schema: BUGS_SCHEMA})
  bugs.push(...result.bugs)
  log(`${bugs.length} found, ${Math.round(budget.remaining()/1000)}k remaining`)
}
```

> ⚠️ **`budget.total` でガードすること**。目標未設定だと `remaining()` は `Infinity` になり、
> ループは 1000 エージェント上限まで走ってしまう。

### 9.5 パターン合成 — 網羅レビュー（find → 既知との重複排除 → 多視点判定 → 枯れるまでループ）

```js
const seen = new Set(), confirmed = []
let dry = 0
while (dry < 2) {                                              // loop-until-dry
  const found = (await parallel(FINDERS.map(f => () =>          // barrier: collect all finders this round
    agent(f.prompt, {phase: 'Find', schema: BUGS})))).filter(Boolean).flatMap(r => r.bugs)
  const fresh = found.filter(b => !seen.has(key(b)))           // dedup vs ALL seen — plain code, not an agent
  if (!fresh.length) { dry++; continue }
  dry = 0; fresh.forEach(b => seen.add(key(b)))
  const judged = await parallel(fresh.map(b => () =>           // every fresh bug judged concurrently...
    parallel(['correctness','security','repro'].map(lens => () =>   // ...each by 3 distinct lenses
      agent(`Judge "${b.desc}" via the ${lens} lens — real?`, {phase: 'Verify', schema: VERDICT})))
      .then(vs => ({ b, real: vs.filter(Boolean).filter(v => v.real).length >= 2 }))))
  confirmed.push(...judged.filter(v => v.real).map(v => v.b))
}
return confirmed
// dedup vs `seen`, NOT `confirmed` — else judge-rejected findings reappear every round and it never converges.
```

**最後のコメントが最重要**です。重複排除は `confirmed` ではなく `seen` に対して行う。そうしないと判定で棄却された findings が毎ラウンド再出現し、永久に収束しません。

### 9.6 敵対的検証

```js
const votes = await parallel(Array.from({length: 3}, () => () =>
  agent(`Try to refute: ${claim}. Default to refuted=true if uncertain.`, {schema: VERDICT})))
const survives = votes.filter(Boolean).filter(v => !v.refuted).length >= 2
```

---

## 10. 品質パターン

仕様が列挙している型です。**タスクに応じて選び、自由に合成する**もので、これが全てではありません。

### 10.1 Adversarial verify（敵対的検証）

各 finding に対し、**反証せよと指示された**独立した懐疑エージェントを N 体立てる。過半数が反証したら棄却。

もっともらしいが誤っている findings が生き残るのを防ぐ。

```js
const votes = await parallel(Array.from({length: 3}, () => () =>
  agent(`Try to refute: ${claim}. Default to refuted=true if uncertain.`, {schema: VERDICT})))
const survives = votes.filter(Boolean).filter(v => !v.refuted).length >= 2
```

ポイント: **「確認してください」ではなく「反証を探せ」**。前者では元の結論に引っ張られる。さらに「不確かなら refuted=true をデフォルトにせよ」と指示することで、判断保留を棄却側に倒す。

### 10.2 Perspective-diverse verify（多視点検証）

finding が複数の壊れ方をしうるとき、N体の同一検証者ではなく、**各検証者に別のレンズを与える**（correctness / security / perf / does-it-reproduce）。

> diversity catches failure modes redundancy can't
> （多様性は、冗長性では捕まえられない失敗モードを捕まえる）

### 10.3 Judge panel（審査パネル）

異なる切り口から N 個の独立した案を生成し（MVP優先・リスク優先・ユーザー優先など）、並列の審査エージェントで採点し、**勝者を軸に次点の良いアイデアを接ぎ木**して統合する。

解空間が広いとき、「1案を反復改善する」より優れる。

### 10.4 Loop-until-dry（枯れるまでループ）

サイズが未知の探索（バグ、課題、エッジケース）では、**K ラウンド連続で新規ゼロ**になるまで finder を回し続ける。

> Simple counters (while count < N) miss the tail.
> （単純なカウンタは末尾を取りこぼす）

### 10.5 Multi-modal sweep（多モード探索）

並列エージェントがそれぞれ**別の探し方**をする（コンテナ別・内容別・エンティティ別・時系列別）。

各エージェントは他が何を見つけたか知らない。1つの検索角度では全部見つからないときに有効。

### 10.6 Completeness critic（完全性批評）

最後に「**何が欠けているか — 実行していないモダリティ、検証していない主張、読んでいない情報源は何か**」だけを問うエージェントを置く。

それが見つけたものが、次のラウンドの作業になる。

### 10.7 No silent caps（黙った打ち切りをしない）

ワークフローが網羅範囲を制限したら（top-N、リトライなし、サンプリング）、**何を落としたか `log()` に残す**。

> silent truncation reads as "covered everything" when it didn't.
> （黙った切り捨ては、実際はそうでないのに「全部やった」と読めてしまう）

---

## 11. ランタイム制約

| 制約 | 値 | 備考 |
|---|---|---|
| 同時実行エージェント数 | **`min(16, CPUコア数 - 2)`** | 超過分はキューされ、スロットが空き次第実行。100件渡しても全部完了はする |
| 生涯エージェント数 | **1000** | 暴走ループのバックストップ。実ワークフローの想定よりはるか上 |
| `parallel()`/`pipeline()` 1回あたりの項目数 | **4096** | 超過はサイレント切り捨てではなく**明示的エラー** |
| 実行環境 | プレーン JavaScript | TypeScript 構文はパースエラー |
| ファイルシステム / Node.js API | **アクセス不可** | I/O はサブエージェントに行わせる |

### 使えない組み込み

標準の JS 組み込み（JSON, Math, Array 等）は使えますが、**次は throw します**:

```js
Date.now()      // ❌ throw
Math.random()   // ❌ throw
new Date()      // ❌ throw （引数なしの場合）
```

理由は **resume（再開）を壊すから**です。

回避策:
- タイムスタンプは `args` 経由で渡す、またはワークフローが返った**後**に付与する
- ランダム性が必要なら、**インデックスでプロンプト／ラベルを変える**

### MCP ツール

ワークフローのエージェントは、セッションに接続された全 MCP ツールに `ToolSearch` 経由でアクセスできます（スキーマはエージェントごとにオンデマンドで読み込まれる）。

> ⚠️ 注意: 対話的に認証する MCP サーバ（例: claude.ai）は、**ヘッドレス／cron 実行では存在しない場合がある**。

---

## 12. Resume（再開）の仕組み

ツール結果に `runId` が含まれます。一時停止・kill・スクリプト編集の後に再開するには:

```js
Workflow({ scriptPath: "<path>", resumeFromRunId: "wf_..." })
```

### 挙動

- **変更されていない `agent()` 呼び出しの最長接頭辞**が、キャッシュから即座に返る
- 最初に編集／追加された呼び出しと、それ以降が実走する
- **同一スクリプト＋同一 args → キャッシュヒット率 100%**
- 同一セッション内でのみ有効。セッションをまたいだ永続再開ではない
- 事前に `TaskStop` で前の実行を止めておくこと

これが `Date.now()` / `Math.random()` を禁止している理由です。

### デバッグ時の鉄則

> Before diagnosing why a completed workflow returned an empty or unexpected result,
> Read `<transcriptDir>/journal.jsonl` — it records each agent's actual return value;
> **do not assume cached results are non-empty.**

`journal.jsonl` に各エージェントの実際の戻り値が記録されています。キャッシュ結果が空でない、と決めつけないこと。

journal が無い場合のフォールバック: transcript ディレクトリの `agent-<id>.jsonl` を読み、継続スクリプトを手書きする。

### 反復のワークフロー

```
1. Workflow({script: "..."})          → scriptPath と runId が返る
2. その scriptPath を Write/Edit で編集
3. Workflow({scriptPath, resumeFromRunId}) → 変更点以降だけ実走
```

全文を再送しない。これが公式の反復手順です。

---

## 13. 規模の決め方と Ultracode

### 依頼の強さに合わせる

> Scale to what the user asked for.

| 依頼 | 構成 |
|---|---|
| "find any bugs" | finder 数体、検証は単一投票 |
| "thoroughly audit this" / "be comprehensive" | finder プールを拡大、3〜5票の敵対的検証、統合ステージ |

迷ったら、**research / review / audit 系は網羅寄り**、クイックチェックは簡潔寄りに倒す。

### サイズガイドライン

セッションには「ワークフローサイズガイドライン」が設定されており、システムプロンプトに動的に注入されます。

```
This session has the default workflow size guideline: medium — keep workflows under 15 agents.
This is a guideline, not a hard limit — follow it unless the user's prompt calls for a different scale.
The user can raise or remove it with "Dynamic workflow size" in /config.
```

**ハードリミットではありません**。ユーザーのプロンプトが別の規模を要求しているなら、それに従います。

### Ultracode

system-reminder で Ultracode が ON と確認できる場合、**オプトインは常時有効**になります。

> author and run a workflow for every substantive task by default.
> The goal is the most exhaustive, correct answer you can produce — token cost is not a constraint.

- 多段の作業（understand → design → implement → review）では、**フェーズごとに複数のワークフローを順に**回す。そうすることで各フェーズ間で人間が介在できる
- 単独で処理してよいのは、会話的なターンか、些末な機械的編集のみ
- reminder が OFF と言えば、通常のオプトイン規則に戻る

### ターンをまたいで連鎖できる単一フェーズ・ワークフロー

| 種類 | 構成 |
|---|---|
| **Understand** | 関連サブシステムへの並列リーダー → 構造化マップ |
| **Design** | N個の独立アプローチを審査パネルで評価 → 採点付き統合 |
| **Review** | 次元分割 → 検出 → 敵対的検証 |
| **Research** | 多モード探索 → 深読み → 統合 |
| **Migrate** | 対象箇所の発見 → 各箇所を変換（worktree 隔離） → 検証 |

大きな作業では、これらを順に走らせる。**各結果を読んでから次のフェーズを決める**。ユーザーは常にループの中に留まり、各ワークフローは1つのよくスコープされた fan-out になります。

---

## 14. 実体はどこにあるか

**結論: ファイルとして存在しません。実行バイナリにコンパイル済みで埋め込まれています。**

### 実行中のバイナリ

```
/Users/<user>/Library/Application Support/Claude/claude-code/2.1.219/claude.app/Contents/MacOS/claude
```

- Mach-O 64-bit arm64、257MB の単一実行ファイル
- デスクトップアプリが `Contents/Helpers/disclaimer` 経由で起動
- 起動引数: `--output-format stream-json --verbose --input-format stream-json --effort xhigh`

### バイト位置（v2.1.219 の場合）

| オフセット | 内容 |
|---|---|
| `234,443,175` | 説明文の先頭 `Execute a workflow script that orchestrates multiple subagents deterministically` |
| `234,443,813` | `ONLY call this tool when the user has explicitly opted into multi-agent orchestration` |
| `234,454,359` | `DEFAULT TO pipeline()` |
| `234,459,577` | `Adversarial verify` |
| `234,460,523` | `Loop-until-dry` |

約 234.44MB〜234.46MB に、**約18KBの連続テキスト**として存在します。

### データではなくソースである証拠

抽出した本文中に、ミニファイ後の変数展開が残っています。

```
For any other task — even one that would clearly benefit from parallelism — do NOT
call this tool. Use the ${Go} tool (if available) for individual subagents, ...
```

`${Go}` は minifier がリネームした「Agent ツール名」の変数。JavaScript のテンプレートリテラルとしてバンドルにコンパイルされたものであり、外部から読み込まれる設定・スキルではありません。

### 3つの落とし穴

1. **`which claude` は当てにならない**
   PATH 上の `claude` は `~/.local/share/claude/versions/<ver>` に解決されるが、デスクトップアプリは Application Support 配下に**別バージョン**を抱えている。実測では CLI が 2.1.214、デスクトップが 2.1.219 だった。

2. **Electron 側には入っていない**
   `/Applications/Claude.app/Contents/Resources/app.asar`（39MB）を grep してもヒット 0。UI シェルとエージェント本体のサイドカーバイナリは完全に分離。

3. **サイズガイドラインだけ別領域から動的合成**
   `keep workflows under 15 agents` という完全文では grep できない。オフセット `103,704,317` 付近（本文とは全く別の領域）に、長さプレフィックス付きの断片として格納されている。

   ```
   "This session has the default workflow size guideline:"
   "A workflow size guideline is configured for this session:"
   " The user can raise or remove it with \"Dynamic workflow size\" in /config."
   ```

### 実際に読む方法

バイナリを掘る必要はありません。**システムプロンプトはセッション transcript に記録されています**。

```bash
grep -o '"description":"Execute a workflow script[^"]*' \
  ~/.claude/projects/<project-slug>/<session-id>.jsonl | head -c 4000
```

これが最も実用的な入手経路です。

---

## 15. 保存場所と実行時アーティファクト

### 保存ワークフロー

| スコープ | パス |
|---|---|
| プロジェクト | `.claude/workflows/` |
| ユーザー | `~/.claude/workflows/` |

ここに置いたものは `Workflow({name: "..."})` や `workflow(name, args)` で呼べます。

### 実行時に自動生成されるもの

```
~/.claude/projects/<project-slug>/<session-id>/
├── workflows/scripts/<meta.name>-<runId>.js     ← 起動のたびに自動保存されるスクリプト
└── subagents/workflows/<runId>/
    ├── journal.jsonl                            ← 各エージェントの実際の戻り値
    └── agent-<id>.jsonl                         ← 個別エージェントのトランスクリプト
```

- **スクリプトファイル**: 反復時はこれを編集して `{scriptPath}` で再実行する
- **journal.jsonl**: 空の結果や想定外の結果を診断するとき、**最初に読むべきファイル**
- **agent-*.jsonl**: journal が無いときのフォールバック

---

## 16. ワークフローを読む・レビューするチェックリスト

JavaScript を開いたら、この順で確認します。

### 構造

1. 最終 `return` は何か（何を返すワークフローなのか）
2. `args` などの入力は何か。外部入力・固定値・エージェント生成値を区別する
3. `agent()` はいくつあるか。各エージェントの役割・入力・出力・**副作用**を表にする
4. 変数を矢印として結び、データフローグラフを復元する
5. `meta.phases` と `phase()` のタイトルは一致しているか

### 並列性

6. **バリアはどこにあるか**。その各バリアは §8 の「正しい場合」に該当するか
7. 独立しているのに逐次 `await` になっている箇所はないか
8. `pipeline()` のステージ内で `opts.phase` を指定しているか（グローバル `phase()` のレースを避けているか）
9. `parallel()` の引数は thunk `() => agent(...)` になっているか

### 正しさ

10. 全ての `agent()` に `schema` があるか
11. schema に**証拠フィールド**があるか。`uncertain` を表現できるか
12. 検証者は実行者から**独立**しているか。「確認せよ」ではなく「**反証せよ**」になっているか
13. 重複排除はどの集合に対して行っているか（`seen` か `confirmed` か — §9.5 参照）

### 失敗とコスト

14. `agent()` が `null` を返しうる箇所で `.filter(Boolean)` しているか
15. `filter(Boolean)` で「失敗」「結果欠損」「意図的スキップ」「指摘なし」を潰していないか
16. ループに**最大回数・成功条件・進展なし停止条件**の3つが揃っているか
17. `budget.total` のガードなしに `budget.remaining()` でループしていないか
18. 生成されるエージェント総数は何体か。ガイドラインの範囲内か
19. 最終集約エージェントに巨大な出力を渡していないか
20. ファイルを書き換えるエージェントはどれか。同時編集競合は起きないか（`isolation: 'worktree'` は必要か）
21. top-N やサンプリングで打ち切っている箇所を `log()` に残しているか

### `meta`

22. `meta` は純粋リテラルか（変数・関数呼び出し・スプレッド・テンプレート補間が無いか）
23. `Date.now()` / `Math.random()` / `new Date()` を使っていないか

---

## 17. アンチパターン

### 17.1 全部を1体のエージェントにやらせる

```js
// ❌
const result = await agent(`
  リポジトリ全体を調査し、問題を探し、修正し、テストし、レポートしてください。
`)
```

何を根拠にどこで失敗したか分からない。`discover → review → verify → fix → test → summarize` に分割する。

### 17.2 不要なバリア

```js
// ❌ 中間の transform に cross-item 依存が無いのにバリアを張っている
const a = await parallel(...)
const b = transform(a)
const c = await parallel(b.map(...))

// ✅ transform をステージ内に入れて pipeline にする
pipeline(items, stageA, r => transform([r]).flat(), stageB)
```

### 17.3 検証者が元エージェントを追認するだけ

```js
// ❌ 元の結論に引っ張られる
agent("この結果が正しいことを確認してください")

// ✅
agent("この主張を反証する証拠を優先して探せ。元の結論を前提にするな。不確かなら refuted=true にせよ。")
```

### 17.4 巨大な共通コンテキストを全エージェントに配る

```js
// ❌ 同じ巨大コンテキストを何度も消費する
const hugeRepositorySummary = "..."
await parallel([
  () => agent(hugeRepositorySummary + taskA),
  () => agent(hugeRepositorySummary + taskB),
  () => agent(hugeRepositorySummary + taskC),
])
```

対象を絞る／エージェントごとに必要な情報だけ渡す／共通知識は Skill や CLAUDE.md へ／探索エージェントに必要箇所を特定させる。

### 17.5 上限のないループ

```js
// ❌
while (!passed) { ... }

// ✅ 最大回数 + 成功条件 + 進展なし停止条件
for (let attempt = 0; attempt < 3; attempt += 1) { ... }
```

### 17.6 `filter(Boolean)` で失敗を消す

```js
const successful = results.filter(Boolean)
```

これだけでは以下を区別できない:

```
null      = agent実行失敗
undefined = 結果欠損
false     = 検証不合格
[]        = 指摘なし
```

schema に `status` を持たせる。

### 17.7 読み取りエージェントと編集エージェントの混在

レビュー担当が勝手に編集すると、後段の検証対象が変化する。

```
探索・レビュー・検証 → 原則読み取り専用
修正               → 明示的に編集許可 + isolation: 'worktree'
テスト             → 修正後の状態を検証
```

### 17.8 巨大すぎる schema

最初から何十項目も持つ schema は、欠損・無意味な埋め草・不必要な出力・トークン増加を招く。各工程が本当に必要とする最小情報にする。

### 17.9 収束しないループ

```js
// ❌ 判定で棄却されたものが毎ラウンド再出現する
const fresh = found.filter(b => !confirmed.some(c => key(c) === key(b)))

// ✅ 見たもの全部に対して重複排除する
const fresh = found.filter(b => !seen.has(key(b)))
```

### 17.10 `args` を文字列化して渡す

```js
// ❌ 1個の文字列として届き、args.map が throw する
Workflow({ script, args: "[\"a.ts\", \"b.ts\"]" })

// ✅
Workflow({ script, args: ["a.ts", "b.ts"] })
```

---

## 18. 実戦例：本ドキュメント執筆セッションで実際に走らせたワークフロー

「マルチエージェント・ワークフロー構築の仕組み」をネット調査するために組んだものです。**6領域を並列調査 → 各領域が終わり次第そのまま独立検証 → 完全性批評 → ギャップ補完 → 統合執筆**という構成。

### 設計の骨格

```
Research  ─ 6領域を並列Web調査
    │       frameworks / patterns / runtime / protocols / claude / ops
    ↓  ※バリアなし（pipeline）— 終わった領域から即座に検証へ流れる
Verify    ─ 各領域の risky な主張を独立エージェントが敵対的に再検証
    ↓  ※ここで初めてバリア（批評には全領域が必要）
Critique  ─ 完全性批評：何が欠けているかを具体的な調査項目として指摘
    ↓
Fill      ─ 上位3件のギャップを追加調査（残件は log() で「未調査」と明示）
    ↓
Synthesize─ 検証結果を反映した統合レポートをファイル出力
```

### 実装（抜粋）

```js
export const meta = {
  name: 'multi-agent-workflow-research',
  description: 'ネット調査: 複数エージェント連携ワークフロー構築の仕組みを網羅調査し、敵対的に検証して統合レポート化する',
  phases: [
    { title: 'Research',   detail: '6領域を並列でWeb調査（Sonnet 5）', model: 'sonnet' },
    { title: 'Verify',     detail: '各領域のリスク主張を独立に再検証',   model: 'sonnet' },
    { title: 'Critique',   detail: '抜けている論点・未検証の主張を洗い出す', model: 'sonnet' },
    { title: 'Fill',       detail: '批評で出たギャップを追加調査',       model: 'sonnet' },
    { title: 'Synthesize', detail: '統合レポートを日本語で執筆しファイル出力', model: 'sonnet' },
  ],
}

// --- 1段目と2段目をバリアなしで繋ぐ ---
const results = await pipeline(
  DIMENSIONS,
  d => agent(researchPrompt(d), {
    label: 'research:' + d.key, phase: 'Research',
    model: 'sonnet', effort: 'high', schema: RESEARCH_SCHEMA,
  }),
  (res, d) => {                                    // ← 第2引数で元の項目を受け取れる
    if (!res) { log('研究失敗のためスキップ: ' + d.key); return null }
    return agent(verifyPrompt(d, res), {
      label: 'verify:' + d.key, phase: 'Verify',
      model: 'sonnet', effort: 'high', schema: VERIFY_SCHEMA,
    }).then(v => ({ dimension: d, research: res, verify: v }))
  }
)

const good = results.filter(Boolean)
log(good.length + '/' + DIMENSIONS.length + ' 領域が調査+検証を完了')

// --- ここからはバリアが正しい：批評は全領域を横断的に見る必要がある ---
phase('Critique')
const critique = await agent(criticPrompt(good), {
  label: 'completeness-critic', phase: 'Critique',
  model: 'sonnet', effort: 'high', schema: CRITIC_SCHEMA,
})

// --- no silent caps: 打ち切った件数を必ず log に残す ---
const allGaps = ((critique && critique.gaps) || []).slice().sort((a, b) => a.priority - b.priority)
const gaps = allGaps.slice(0, 3)
if (allGaps.length > gaps.length) {
  log('批評家が指摘したギャップ ' + allGaps.length + ' 件のうち、上位 ' + gaps.length + ' 件のみ追加調査（残りは未調査として報告）')
}
```

### 採用した設計判断とその理由

| 判断 | 理由 |
|---|---|
| Research→Verify を `pipeline` にした | 検証は領域ごとに独立。バリアを張ると速い領域のアイドル時間を捨てる |
| Critique の前だけ `parallel`（暗黙のバリア） | 批評は「領域間で矛盾していないか」を見るので、全結果が同時に必要 |
| 検証エージェントに `risky` フラグ付きの主張だけ渡す | 全件検証はコストが釣り合わない。誤ると実害が出る主張（バージョン、GA状態、数値、API名）に絞る |
| 検証プロンプトを「元調査員を信用するな」から始めた | 追認バイアスを避ける。§10.1 |
| `CONFIRMED / CORRECTED / REFUTED / UNVERIFIABLE` の4値 | 二値だと「判断つかない」が CONFIRMED に流れ込む |
| ギャップ補完を上位3件に制限し、`log()` で明示 | §10.7 no silent caps |
| 全エージェントに「カットオフの記憶で書くな、検索8回以上」と明示 | Web調査エージェントの最大の失敗モードは、検索せず記憶で答えること |

### プロンプト側の工夫

全エージェント共通のルールブロックを定数にして各プロンプトへ差し込みました。

```js
const COMMON = [
  '## 厳守ルール',
  '- 今日は' + TODAY + '。あなたの学習データのカットオフは2026年5月です。**記憶だけで書くことを固く禁じます。**',
  '- 必ず WebSearch / WebFetch を使い、一次情報に当たること。',
  '- 検索は最低8回、深掘りの WebFetch は最低5ページ行うこと。1回検索して終わりにしない。',
  '- 製品名・機能名・API名・バージョン番号・数値は、実際にページで確認できたものだけ書く。',
  '  確認できなければ confidence を "unverified" にし、claim 中にも「未確認」と明記する。',
  '- ありそうな機能名を捏造しないこと。「見つからなかった」と正直に書くほうが遥かに価値が高い。',
  '- 事実と、あなたの解釈・評価を明確に分けること。解釈には「解釈:」と前置きする。',
  '- 出力は日本語。あなたの最終出力は人間へのメッセージではなく、後続処理が使う構造化データです。',
].join('\n')
```

最後の一文が効きます。サブエージェントは放っておくと人間向けの体裁で返そうとします。

---

## 19. 周辺機構との関係

Workflow は唯一の連携手段ではありません。使い分けの一覧です。

| 機構 | 誰が次の処理を決めるか | 状態の持ち方 | 向いている用途 |
|---|---|---|---|
| **Workflow** | **JavaScript** | JS 変数 | 再現可能・決定論的な多段 fan-out |
| `Agent` ツール | 呼び出し元の Claude | 各エージェントの独立コンテキスト | 単発の委譲、探索 |
| `SendMessage` | エージェント同士 | 各自のトランスクリプト | 常駐チームとの往復。名前指定で会話状態を保ったまま再開 |
| Task ボード（`TaskCreate` 等） | 各ワーカーが claim | 共有ボード（`owner` / `blockedBy` / `metadata`） | 依存関係つきの共有ワークキュー |
| Skill (`SKILL.md`) | Claude | — | 手順・規約のパッケージ化 |
| Hooks (`settings.json`) | **ハーネス（事前定義ルール）** | — | モデル判断に依存しない決定論的な前後処理 |
| `Monitor` | 外部イベント | — | ログ・CI・WebSocket の監視。1行=1通知 |
| cron / scheduled-tasks | 時刻 | 毎回ゼロから | 定期実行。プロンプトは自己完結必須 |

### Workflow と Agent ツールの境界

- **1体だけ委譲したい** → `Agent`
- **同じ処理を N 件に適用したい / 多段で検証したい / 決定論的なループが要る** → `Workflow`
- **会話状態を保ったまま何度も往復したい** → `Agent` で名前付き起動 → `SendMessage`

### Workflow と Task ボードの併用

Workflow の中では JS 変数が状態なので、Task ボードは不要です。逆に、**ユーザーに進捗を見せたい**場合や、**ワークフローの外側で長期に渡る作業を追跡する**場合には Task ボードが有効です。

---

## 20. 確認済み事項と未確認事項の区別

### このドキュメントで確認済みの事項

以下はすべて、実行中バイナリに埋め込まれた `Workflow` ツール仕様の記述、またはこのマシン上での実測に基づきます。

- §2 オプトイン条件（5項目）
- §4 `meta` の必須フィールドと純粋リテラル制約
- §5 全フック（`agent` / `pipeline` / `parallel` / `log` / `phase` / `args` / `budget` / `workflow`）の挙動
- §6 `agent()` の全オプションと、`model` を省略すべきという指針
- §8 バリア原理、スメルテスト
- §9 コード例（全て仕様からの逐語引用）
- §10 品質パターン7種
- §11 制約値: 並行度 `min(16, コア数-2)`、生涯1000体、1回4096件、`Date.now()`/`Math.random()`/`new Date()` が throw すること
- §12 resume の接頭辞キャッシュ、`journal.jsonl` を先に読むべきこと
- §13 サイズガイドラインが動的注入であること、Ultracode の挙動
- §14 バイナリのパス・バイトオフセット・`${Go}` の存在（実測）
- §15 保存パスと実行時アーティファクトのパス（実測）

### 未確認 / 推測を含む事項

- **バージョン間の差異**: ここに書いた内容は v2.1.219 で確認したもの。他バージョンでの一致は未確認
- **`agentType` で指定できる型の完全な一覧**: 「`Agent` ツールと同じレジストリから解決される」とだけ書かれており、具体的な一覧は環境依存
- **`effort` の各値が実際にどの程度の推論量に対応するか**: 定量的な記述は無い
- **`whenToUse` フィールドの正確な表示箇所**: 「ワークフロー一覧に表示される」とのみ
- **§19 の周辺機構との比較表**: 各機構の仕様は確認済みだが、「向いている用途」列は解釈
- **§16 チェックリストと §17 アンチパターン**: 仕様の記述を基にした実務上の整理であり、一部は解釈を含む

---

## 付録: 最小の動くテンプレート

```js
export const meta = {
  name: 'my-workflow',
  description: '一行で何をするか',
  phases: [{ title: 'Find' }, { title: 'Verify' }],
}

const ITEM_SCHEMA = {
  type: 'object',
  additionalProperties: false,
  properties: {
    items: {
      type: 'array',
      items: {
        type: 'object',
        additionalProperties: false,
        properties: {
          title:    { type: 'string' },
          evidence: { type: 'string' },
        },
        required: ['title', 'evidence'],
      },
    },
  },
  required: ['items'],
}

const VERDICT_SCHEMA = {
  type: 'object',
  additionalProperties: false,
  properties: {
    refuted: { type: 'boolean' },
    reason:  { type: 'string' },
  },
  required: ['refuted', 'reason'],
}

const TARGETS = ['a', 'b', 'c']

const results = await pipeline(
  TARGETS,
  t => agent(`${t} を調べて items を返せ。証拠を必ず添えること。`, {
    label: `find:${t}`, phase: 'Find', schema: ITEM_SCHEMA,
  }),
  (found, t) => {
    if (!found) { log(`スキップ: ${t}`); return null }
    return parallel(found.items.map(it => () =>
      agent(`次を反証せよ。不確かなら refuted=true にせよ: ${it.title} / 根拠: ${it.evidence}`, {
        label: `verify:${t}:${it.title.slice(0, 20)}`, phase: 'Verify', schema: VERDICT_SCHEMA,
      }).then(v => ({ ...it, target: t, survived: v ? !v.refuted : false }))
    ))
  }
)

const confirmed = results.flat().filter(Boolean).filter(x => x.survived)
log(`${confirmed.length} 件が検証を通過`)

return { confirmed, checked: results.flat().filter(Boolean).length }
```

このテンプレートに含まれている要素:

- `meta` は純粋リテラル
- `pipeline` でバリアなし（Find が終わった target から Verify へ）
- ステージ2で `(prevResult, originalItem)` の第2引数を使用
- `agent()` が `null` を返す可能性への対処（2箇所）
- 「確認せよ」ではなく「反証せよ」＋「不確かなら refuted=true」
- `label` に対象を含める
- `phase` をステージ内で明示指定
- schema に証拠フィールドあり
- `log()` で結果件数を明示
