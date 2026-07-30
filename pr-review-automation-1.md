> From: https://chatgpt.com/c/6a66d603-4cd8-83ee-99fe-53c2200cfbe2

# you asked

message time: 2026-07-27 12:52:34

GithubにおけるPRの自動レビューBotの性能を上げるための手法について、既存な様々なサービスや知見を調査し説明してください。さらに、それらを結集して最高のシステムを考えてください。

---

# chatgpt response

I’ll compare current services and primary research across LLM review, static analysis, security, and operations, then design an implementable layered system with execution validation, repository history, feedback, metrics, and phased rollout.

---

# chatgpt response

Benchmarks still show low detection of human-identified issues, so gains depend on selective context retrieval, reviewer ensembles, and candidate-comment revalidation rather than indiscriminately expanding prompts.

---

# chatgpt response

# GitHub PR自動レビューBotの性能を最大化する設計

## 結論

2026年7月時点で、最も性能の高いPRレビューシステムは、単一のLLMに差分を渡してコメントさせるBotではありません。次の多層パイプラインです。

```text
PR取得
  ↓
信頼境界・リスク判定
  ↓
ビルド／型検査／静的解析／既存テスト
  ↓
コード・依存関係・仕様・過去PRの関連文脈を抽出
  ↓
専門エージェントによる並列レビュー
  ↓
テスト生成・base/head比較・到達可能性解析による裏取り
  ↓
Judgeによる統合・重複排除・誤検出除去
  ↓
ポリシー判定
  ↓
少数の証拠付きコメントとGitHub Checkを公開
  ↓
修正・却下・障害情報を評価データへ還元
```

最重要点は、**検出候補を増やすことより、候補を検証し、誤検出とノイズを公開前に落とすこと**です。

独立研究では、実際の人間レビューを基準にした場合、最先端モデルでも検出できる問題は限定的です。SWE-PRBenchでは8つのフロンティアモデルが、人間の指摘の15〜31%程度しか検出できず、単純にリポジトリ全文を追加すると、むしろ成績が低下する傾向が報告されています。c-CRABでも既存レビューエージェントを合わせた解決率は約40%でした。一方、複数レビューの統合、候補生成と検証の分離、actionability判定は明確に性能を改善しています。([arXiv](https://arxiv.org/html/2603.23448v2))

したがって、最適化対象はコメント数ではなく、概念的には次です。

```text
レビュー価値
= 重大な真の問題を発見した価値
- 誤検出を人間が確認するコスト
- 重複・古いコメントのコスト
- 見逃しによる障害コスト
- 実行時間・LLM・CIコスト
```

---

# 1. 既存サービスから学べること

以下の機能説明は主に各社の公式資料に基づきます。各社のマーケティング上の性能値は、条件が異なるため直接比較できません。

## GitHub Copilot Code Review / GitHub Code Quality

GitHub Copilot Code Reviewは、自動レビュー、pushごとの再レビュー、リポジトリ共通・パス別のカスタム指示、`AGENTS.md`、`REVIEW.md`、MCP、カスタムセットアップワークフローなどを利用できます。ただしCopilotレビュー自体は、PRにCommentを残すだけで、ApproveやRequest changesは行わず、required approvalにも算入されません。([GitHub Docs](https://docs.github.com/copilot/using-github-copilot/code-review/using-copilot-code-review))

2026年7月20日に一般提供されたGitHub Code Qualityは、CodeQLによる決定的解析とAI支援の品質検出、修正提案、カバレッジ、rulesetによる品質ゲートをGitHub内で統合しています。GitHub TeamまたはEnterprise Cloudで利用でき、GitHubネイティブな基盤として非常に強力です。([The GitHub Blog](https://github.blog/changelog/2026-07-20-github-code-quality-is-now-generally-available/))

**学ぶべき点**

- レビューコメントとマージゲートを分離する
- リポジトリ固有ルールをコードと一緒に管理する
- GitHub Checks、ruleset、merge queueまで統合する
- LLMだけでなくCodeQLなどの決定的解析を基礎に置く

---

## CodeRabbit

CodeRabbitは、リポジトリをサンドボックス内にcloneし、多数の静的解析ツールと複数の専門エージェントを組み合わせ、コードベース探索、検証、レビュー統合、過去のフィードバックからの学習を行う構成を採用しています。ベンダー資料では50以上のリンター・静的解析・SASTとの統合を説明しています。([CodeRabbit](https://docs.coderabbit.ai/overview/architecture))

**学ぶべき点**

- LLMレビューと既存解析器を同じパイプラインに入れる
- PRの差分だけでなく、実際のリポジトリを安全な環境で調査する
- Review AgentとVerification Agentを分離する
- 過去に受け入れられた指摘を継続的なルールとして蓄積する

---

## Qodo Merge

Qodoは、correctness、security、architecture、standardsなどの専門エージェントを並列に動かし、最後にJudge Agentが重複除去、矛盾解決、信頼度・関連性によるフィルタリングを行います。過去のPR、チームの対応、組織ルール、コードベース、クロスリポジトリ関係をコンテキストとして利用する設計です。([Qodo Documentation](https://docs.qodo.ai/core-concepts/qodo-platform-core-capabilities))

**学ぶべき点**

- 複数エージェントを直接PRに書き込ませない
- 最終的な一人のJudgeまたはFinalizerを置く
- 「以前も同じ指摘をしたか」「その指摘は実際に修正されたか」を見る
- 組織ルールを中央管理し、各リポジトリへ適用する
- APIやスキーマ変更では、利用側リポジトリまで調べる

なお、オープンソースのPR-Agentは現在、Qodoの主力製品とは別のコミュニティ管理プロジェクトです。自作システムの土台としては有用ですが、そのまま最高性能の製品と考えるべきではありません。([GitHub](https://github.com/The-PR-Agent/pr-agent))

---

## Greptile

Greptileは、ファイル、関数、依存関係をグラフとして索引化し、複数のエージェントが並列にコードベースを探索する方式を採っています。また、サンドボックス内でテストを作成・実行する機能も提供しています。([Greptile](https://www.greptile.com/))

**学ぶべき点**

- 通常のテキスト検索だけでなく、コードグラフを持つ
- import、call、schema、routeなどの関係をレビューに使う
- 指摘を文章で終わらせず、再現テストへ変換する

---

## Graphite Reviewer / Cursor Bugbot

Graphiteは、パス別ルール、生成物の除外、PRのフィルタリングに加え、指摘数、修正につながった指摘、受諾率、upvote/downvoteなどを測定できます。受諾率の低いルールを調整・削除する運用が想定されています。([Graphite](https://graphite.com/docs/ai-review-customization))

Cursor Bugbotも、リポジトリ固有ルール、`BUGBOT.md`、自動修正、PRフィードバックからのルール学習、追加pushに対する差分レビューを強化しています。([Cursor](https://cursor.com/docs/bugbot))

**学ぶべき点**

- ルールごとの精度を計測する
- 生成ファイル、lock file、vendor codeを通常レビューから除外する
- pushのたびにPR全体を再コメントせず、新しい変更だけを見る
- チームの却下理由をルール改善に利用する

---

## CodeQL、Semgrep、SonarQube、Snyk、Dependency Review

これらはLLMレビューの競合ではなく、基礎となる検出器です。

- CodeQLはコードをデータベース化し、データフローや脆弱性クエリを実行できます。標準クエリだけでなく、組織固有クエリも追加できます。([GitHub Docs](https://docs.github.com/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning-with-codeql))
- Semgrepは多数の言語に対応した静的解析とカスタムルールを提供し、AIによるトリアージとも組み合わせられます。([GitHub](https://github.com/semgrep/semgrep))
- SonarQubeは特に「今回追加された新しいコード」に対するQuality Gateを設ける考え方が強いです。([SonarSource Docs](https://docs.sonarsource.com/sonarqube-server/discovering/code-analysis/pull-request-analysis))
- GitHub Dependency Reviewは、PRで追加・更新された直接・間接依存関係、脆弱性、ライセンスを比較し、ポリシー違反をrequired checkとして止められます。([GitHub Docs](https://docs.github.com/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review))

LLMに「SQL injectionがないか全部考えて」と頼むより、CodeQLやSemgrepで候補を抽出し、LLMには到達可能性、実際の影響、修正方法を調査させる方が安定します。

---

# 2. 研究・実運用から分かる性能向上手法

## 2.1 「リポジトリ全体を知る」と「全文をプロンプトに入れる」を区別する

多くのサービスは「コードベース全体を理解する」と説明します。しかし、正しい実装は次です。

- リポジトリ全体を索引・グラフ化しておく
- PRごとに必要な部分だけを選び出す
- 選ばれた最小限の文脈だけをモデルへ渡す

SWE-PRBenchでは、差分に周辺ファイルやリポジトリ全文を単純追加すると、モデルの性能が悪化しました。大量の無関係コードが重要な差分を埋もれさせ、ファイル間関係も明示されないためです。構造化された約2,000トークンの差分要約が、より大きな非構造化コンテキストを上回る例も報告されています。([arXiv](https://arxiv.org/html/2603.26130v1))

理想的なContext Packは次を含みます。

```text
1. PR差分
2. 変更された関数・型・クラスの完全な定義
3. 直接のcaller / callee
4. 影響するテスト
5. 関連する設定・ルーティング・スキーマ
6. 関連するADR・仕様・issue
7. 同じ箇所に関する過去のPR判断
8. 外部リポジトリの利用箇所
```

Echoの研究も、変更箇所を中心にした「必要十分な依存関係の閉包」を取得し、テスト実行と検証へ接続する設計を提案しています。([arXiv](https://arxiv.org/html/2603.07326v1))

---

## 2.2 決定的解析を先に実行する

レビューの最初に次を実行します。

```text
build
typecheck
lint
format check
unit / integration tests
CodeQL / Semgrep
dependency / license / secret scan
schema / migration checks
coverage diff
```

その結果をLLMへ渡し、LLMには以下を担当させます。

- エラーの根本原因を説明する
- 複数の警告を一つの問題にまとめる
- 到達可能か、実害があるかを判断する
- 修正候補を生成する
- 不足しているテストを作る

これにより、LLMがコンパイラや静的解析器の劣化版になることを防げます。

---

## 2.3 Candidate GeneratorとVerifierを分離する

一つのモデルに「問題を見つけ、正しいか判断し、コメントも書け」と頼むと、自分の推測を自分で正当化しやすくなります。

より良い構成は次です。

```text
Generator:
  高い再現率を狙い、多めに問題候補を出す

Verifier:
  各候補について、位置・到達可能性・再現性・新規性を確認する

Judge:
  複数候補を統合し、公開すべきものだけを選ぶ
```

ByteDanceのBitsAI-CRでは、RuleCheckerの後にReviewFilterを設ける二段構成により、精度を約60%から最大75%へ改善したと報告されています。また「最初に結論を判定してから説明を生成する」フィルタが、直接コメントさせる方式より高精度でした。([arXiv](https://arxiv.org/html/2501.15134v1))

Atlassianの大規模社内運用でも、候補生成後に「事実として正しいか」と「開発者が行動可能か」を判定する構成が採用されています。特にactionability gateは、人間の判断との一致を15ポイント改善しました。([arXiv](https://arxiv.org/html/2601.01129v2))

---

## 2.4 複数レビューを統合する

SWR-Benchでは、同一モデルまたは異なるモデルによる複数レビューを集約・統合するだけでもF1が最大43.67%改善しました。ただし、候補を単に連結すると誤検出も増えるため、Judgeと検証が必要です。([arXiv](https://arxiv.org/html/2509.01494v1))

専門エージェントは、たとえば次のように分けます。

- Correctness・境界条件
- Security・権限・データフロー
- Concurrency・状態遷移
- Performance・計算量・I/O
- API・スキーマ・後方互換性
- Migration・データ保全
- Tests・observability
- UI・accessibility
- Architecture・組織固有ルール

すべてのPRに全エージェントを使う必要はありません。変更内容に応じて選択します。

---

## 2.5 「baseでは成功、headでは失敗」を証明する

非常に強い証拠は、次の二重実行です。

```text
PR適用前のbase SHA  → テスト成功
PR適用後のhead SHA  → 同じテストが失敗
修正候補適用後      → テスト成功
```

これにより、次を判別できます。

- PRが新たに導入した問題か
- 以前から存在した問題か
- テスト自体が壊れていないか
- 提案修正が本当に有効か

Echoは、テスト生成、実行、検証、変更前後の比較を組み合わせる設計を採用しています。([arXiv](https://arxiv.org/html/2603.07326v1))

実行できない問題では、代わりに次を証拠とします。

- CodeQL/Semgrepのdata-flow path
- callerから問題箇所までのcall path
- schema compatibility差分
- public APIの利用側一覧
- 型・設定・ルーティング間の不整合
- 既存仕様・ADRとの矛盾

---

## 2.6 コード以外のグラフも見る

大規模リポジトリでは、コードがコンパイル・テストを通っても、リポジトリ全体の構造上の不変条件を破ることがあります。

必要になるグラフは次です。

```text
import graph
call graph
data-flow graph
package dependency graph
configuration graph
database schema graph
route graph
resource / permission graph
event / state-transition graph
cross-repository contract graph
```

2026年のPatchwork Problemの研究では、LLM生成コードが局所的には正しくても、設定、依存、スキーマ、ルート、リソースなど複数グラフをまたぐ制約を破る問題が整理されています。まだ新しい研究ですが、巨大システムのレビューでは重要な方向性です。([arXiv](https://arxiv.org/html/2607.08981v1))

---

## 2.7 ルールを短く、具体的に、パス別にする

長大な「チームのコーディング方針」を毎回すべて渡すより、変更パスと問題カテゴリに応じて必要なルールだけを取得します。

よいルールの形式は次です。

```yaml
id: AUTH-004
paths:
  - "src/auth/**"

rule: |
  認可判定は、リクエストで渡されたuserIdではなく、
  認証済みセッションのsubjectを基準に行う。

bad_example: |
  canEdit(req.body.userId, document)

good_example: |
  canEdit(session.subject, document)

reason: |
  クライアント入力を権限主体として信頼すると、
  他ユーザーへのなりすましが可能になる。

severity: critical
required_evidence:
  - data_flow
  - reachable_call_path
```

GitHub自身も、AIの出力は非決定的であり、長い指示は見落とされる可能性があるため、明確で短く具体的なルールを推奨しています。GraphiteもRule、Bad example、Good example、Reasoningという構造を推奨しています。([GitHub Docs](https://docs.github.com/en/copilot/tutorials/customize-code-review))

---

## 2.8 小さいPRを促進する

大きなPRを強いモデルで無理に読むより、変更を意味のある単位に分割した方が、人間にもBotにも有効です。

GoogleのEngineering Practicesでも、小さく自己完結した変更は、レビューが速く、丁寧になり、バグが入りにくく、ロールバックもしやすいとされています。([Google GitHub](https://google.github.io/eng-practices/review/developer/small-cls.html))

ただし、LOCだけで機械的にブロックするのは適切ではありません。Botは次を提示する方が有益です。

- 独立して分割できる変更群
- 仕様変更とリファクタリングの混在
- 自動生成差分の分離
- migrationとアプリ変更の適切な順序
- レビュー対象外にできるファイル

---

# 3. 統合した「最高のシステム」

ここでは、これを **Evidence-Driven Review Fabric** と呼びます。

## 3.1 全体構成

```text
┌──────────────── GitHub ────────────────┐
│ PR / push / review / merge_group event │
└──────────────────┬─────────────────────┘
                   ↓
┌────────────────────────────────────────┐
│ 1. Snapshot & Trust Classifier         │
│ base/head固定、fork判定、変更者、権限   │
└──────────────────┬─────────────────────┘
                   ↓
┌────────────────────────────────────────┐
│ 2. Risk Router                         │
│ 変更範囲、重要度、波及範囲、新規性      │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 3. Secure Analysis Sandbox             │
│ build, lint, test, CodeQL, Semgrep...  │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 4. Repository Intelligence Graph       │
│ AST/LSP/call/config/schema/PR history  │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 5. Context Router                      │
│ 最小限の関連コード・仕様・履歴を取得    │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 6. Specialist Reviewers                │
│ correctness/security/API/test/...      │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 7. Evidence Verifier                   │
│ 再現テスト、base/head比較、到達可能性   │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 8. Judge / Deduplicator / Policy Gate  │
│ 統合、重複排除、優先度、ブロック判定    │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 9. Single Publisher                    │
│ Check Run + 少数のinline comment       │
└──────────────┬─────────────────────────┘
               ↓
┌────────────────────────────────────────┐
│ 10. Feedback & Evaluation Store        │
│ fixed/dismissed/reverted/incidents     │
└────────────────────────────────────────┘
```

---

## 3.2 PRリスク判定

すべてのPRに同じ計算量を使わず、次の特徴からリスクを計算します。

```text
risk =
  w1 × change_surface
+ w2 × criticality
+ w3 × blast_radius
+ w4 × novelty
+ w5 × uncertainty
```

### Change surface

- 変更行・ファイル・パッケージ数
- public API変更
- データベースmigration
- 新規設定や依存関係

### Criticality

- 認証・認可
- 支払い
- 暗号・秘密情報
- データ削除
- CI/CD
- 権限・サンドボックス
- 永続化・監査ログ

### Blast radius

- caller数
- 利用サービス数
- クロスリポジトリ利用
- production pathへの到達可能性

### Novelty

- 新しいライブラリ
- 新しいアーキテクチャパターン
- 既存例のない実装
- 新しい外部API

### Uncertainty

- テスト不足
- 仕様・issueとのリンク不足
- 型の弱い境界
- 動的設定
- 大量の生成コード

---

## 3.3 リスク別の処理

| リスク | 実行するレビュー | 公開方針 |
|---|---|---|
| 低 | 決定的解析＋軽量モデル1つ | 原則サマリーのみ |
| 中 | 関連コンテキスト＋専門エージェント | 検証済みのinlineコメント |
| 高 | 複数モデル＋テスト生成＋依存グラフ | required check候補 |
| 最重要 | クロスリポジトリ解析＋脅威モデル＋人間の専門reviewer | 必須承認・厳格なゲート |

この方式なら、単純なドキュメント修正に高価なモデルを何度も使わず、認証・migration・権限境界に計算資源を集中できます。

---

## 3.4 Findingを文章ではなく構造化データにする

各エージェントには自由形式コメントではなく、共通スキーマを返させます。

```json
{
  "finding_id": "AUTH-004:src/api/document.ts:updateDocument",
  "category": "authorization",
  "severity": "critical",
  "confidence": 0.96,
  "base_sha": "abc123",
  "head_sha": "def456",
  "introduced_by_pr": true,
  "location": {
    "path": "src/api/document.ts",
    "line": 123,
    "symbol": "updateDocument"
  },
  "claim": "リクエスト由来のuserIdを権限主体として利用している",
  "evidence": [
    {
      "type": "data_flow",
      "value": "request.body.userId -> canEdit()"
    },
    {
      "type": "call_path",
      "value": "PATCH /documents/:id -> updateDocument -> canEdit"
    },
    {
      "type": "dual_version_test",
      "command": "pnpm test auth-regression",
      "base": "pass",
      "head": "fail"
    }
  ],
  "rule_refs": ["AUTH-004"],
  "suggested_patch": "...",
  "patch_validation": {
    "build": "pass",
    "tests": "pass"
  },
  "blocking": true
}
```

このデータを `EvidenceBundle` として扱えば、Judgeは複数モデル・静的解析器・テスト結果を同じ形式で統合できます。

---

## 3.5 Judgeの優先度計算

概念的には次のスコアを使います。

```text
priority =
  impact
× calibrated_confidence
× reachability
× introduced_by_pr
× actionability
```

重要なのは、`confidence: 0.96`をモデルの自己申告にしないことです。カテゴリ・言語・モデルごとに、過去の検証結果から校正します。

たとえば、次の条件を満たすものだけをブロック対象にします。

```text
severity >= high
AND introduced_by_pr = true
AND exact_location_verified = true
AND (
  deterministic_evidence exists
  OR base/head reproduction exists
)
AND calibrated_precision >= blocking threshold
```

**LLMの文章上の推測だけでマージを止めてはいけません。**

---

## 3.6 公開コメントを絞る

内部では20件の候補を持っていても、PRに20コメント投稿する必要はありません。

初期方針としては次が適切です。

- Critical/Highかつ検証済み：inline comment
- Mediumかつ修正可能：inlineまたはCheck summary
- 低信頼・設計上の相談：summary
- style/nit：既存formatterに任せる
- 重複・既知・変更前から存在：非公開
- 1 PRあたりinlineコメントは原則3〜5件まで
- 同じ問題の複数箇所は一つの代表コメントと一覧にまとめる

Googleのレビュー指針でも、技術的事実と好みを区別し、非ブロッキングなnitを明確に分けることが推奨されています。([Google GitHub](https://google.github.io/eng-practices/review/reviewer/standard.html))

---

## 3.7 コメントの状態を管理する

各Findingには状態機械を持たせます。

```text
detected
  → verified
  → published
  → acknowledged
  → fixed
      または dismissed(reason)
      または outdated
      または duplicate
```

pushごとに新しいコメントを増やすのではなく、次のfingerprintで既存Findingと照合します。

```text
fingerprint =
  rule_id
+ normalized_symbol
+ semantic_location
+ evidence_signature
```

行番号が変わっても同じ関数・同じ原因なら同一Findingとして追跡します。

---

# 4. セキュリティ上の必須設計

PRレビューBotは、攻撃者が作成したコード・コメント・設定を読むため、通常のチャットBotより厳しい信頼境界が必要です。

## 4.1 外部PRのコードを特権環境で実行しない

GitHubは、`pull_request_target`で外部PRのコードをcheckout・実行すると、write権限のtokenやsecretが盗まれる危険があると警告しています。推奨されるのは次の二段構成です。([GitHub Security Lab](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/))

```text
非特権 workflow:
  pull_request
  read-only token
  secretsなし
  PRコードを実行
  結果を受動的artifactとして保存

特権 workflow / GitHub App:
  workflow_runまたはWebhook
  artifactをデータとして読む
  絶対に実行しない
  Check/commentを投稿
```

さらに、GitHub Actionsでは、`GITHUB_TOKEN`を最小権限にし、外部Actionを完全なcommit SHAに固定し、`.github/workflows`をCODEOWNERSで保護することが推奨されています。([GitHub Docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions))

---

## 4.2 PR内の指示ファイルを信頼しすぎない

GitHub Copilot Code Reviewは、PRのhead branchにある指示ファイルを読みます。([GitHub Docs](https://docs.github.com/copilot/using-github-copilot/code-review/using-copilot-code-review))

自作システムでは、これは次のように扱うべきです。

```text
信頼済みポリシー:
  protected base branch
  中央ガバナンスストア
  管理者が承認したルール

信頼されない補助情報:
  PR head branchのAGENTS.md
  REVIEW.md
  ソースコード内コメント
  issue本文
  PR本文
```

head branchの情報はレビュー対象のデータとして参照しても、次を変更させてはいけません。

- system prompt
- ツール権限
- network allowlist
- secretへのアクセス
- blocking threshold
- 解析対象外パス
- 組織セキュリティルール

---

## 4.3 ツール実行をallowlist化する

エージェントが自由なshell commandを実行する方式ではなく、Capability Manifestを設けます。

```yaml
capabilities:
  filesystem:
    read: repository
    write: temporary_workspace
  network:
    mode: deny_by_default
    allow:
      - registry.npmjs.org
      - crates.io
  commands:
    - pnpm
    - cargo
    - git
    - semgrep
    - codeql
  secrets: none
  timeout_seconds: 900
  cpu_limit: 4
  memory_limit_mb: 8192
```

外部PRでは依存関係のinstall scriptにも注意し、可能ならlockfile、キャッシュ、隔離されたVMを使用します。

---

# 5. ShioriとKanadeに組み込む場合の最適設計

あなたが進めているShioriとKanadeの分離は、このレビューシステムと非常に相性がよいです。

## Shiori：Review Governance Plane

Shioriには、人間が承認した正本だけを置きます。

```text
ReviewPolicy
ReviewRule
ADR
Specification
ArchitectureInvariant
AcceptedFinding
DismissalReason
EvalCase
ProviderPolicy
RiskBudget
BlockingPolicy
```

Shioriが担当するものは次です。

- レビュールールの提案・承認・廃止
- 組織共通ルールとリポジトリ固有ルールの管理
- ADR・仕様・過去判断の正本化
- 誤検出・却下理由のレビュー
- モデル・ルールごとの精度ダッシュボード
- blocking policy変更の人間承認
- ベンチマークケースの正本管理

PRのhead branchにある記述より、Shioriで承認されたポリシーを優先します。

---

## Kanade：Review Execution Plane

Kanadeは実行・調査・検証を担当します。

```text
GitHub webhook intake
ReviewRun scheduler
Risk Router
Sandbox manager
Repository graph builder
Context Router
Provider adapters
Static analyzer adapters
Test-generation agent
Evidence Verifier
Judge / Finalizer
GitHub Check publisher
Audit/Event log
Budget and permission enforcement
```

Codex、Claude Code、GitHub Copilotなどは、Kanadeから見れば交換可能なReviewer Adapterにします。

```text
ReviewerAdapter
├── capability_manifest()
├── review(context_pack, policy) -> Finding[]
├── verify(finding, evidence) -> VerificationResult
├── generate_test(finding) -> TestPatch
└── propose_fix(finding) -> CodePatch
```

`codex exec --json --output-schema`のような構造化実行は、最初の実装として非常に適しています。ただし、Codex単独を真実の源にはせず、CodeQL、テスト、別モデル、Judgeによって検証します。

---

## 単一のFinalizerだけがGitHubへ書き込む

CodeQL、Semgrep、Codex、Claude、テストエージェントなどが直接コメントすると、重複と矛盾が増えます。

```text
各ツール
  ↓ Finding / Evidenceのみ提出
Kanade Finalizer
  ↓ 統合・重複排除・ポリシー判定
GitHub
  ↓ Check Run 1件 + 必要最小限のinlineコメント
```

GitHub Checks APIは、Check Run、詳細な結果、行へのannotationを提供でき、branch protectionやmerge queueのrequired checkとも組み合わせられます。([GitHub Docs](https://docs.github.com/rest/checks))

---

## 保存すべき監査情報

各ReviewRunについて、最低限次を保存します。

```text
repository / PR number
base SHA / head SHA
trust classification
risk score and reasons
policy version
rule versions
model/provider/version
prompt/template hash
tool versions
context document IDs and hashes
generated findings
verification evidence
published comments
human disposition
execution time
token/cost usage
final merge/revert/incident outcome
```

これにより、「モデルが変わったから性能が変わった」のか、「ルール・検索・検証が変わったから改善した」のかを分析できます。

---

# 6. Shioriリポジトリ向けの具体的レビュー

Next.js、TypeScript、Tauri、SQLite、Drizzleを想定すると、次の構成が適しています。

## 常時実行

```text
TypeScript typecheck
ESLint
unit tests
dependency review
CodeQL / Semgrep
secret scan
actionlint
Rust fmt / clippy / test
```

## 変更内容に応じた選択実行

### UI・ルート変更

- 対象画面だけPlaywrightを実行
- console error、404、hydration errorを確認
- screenshot差分
- keyboard操作・accessibility
- server/client component境界
- 権限のある画面が直接URLで開けないか

### SQLite・Drizzle migration

- 空DBへの全migration
- 既存スナップショットDBへのupgrade
- rollback可能性
- destructive column/table change
- non-null追加時のdefault/backfill
- index・foreign key不足
- migration途中で失敗した場合の整合性
- アプリ旧版とDB新版の互換期間

### MCP・エージェント機能

- tool schemaの後方互換性
- read/write capabilityの取り違え
- untrusted contentによるtool invocation
- canonical stateを直接変更していないか
- human approval gateを迂回していないか
- audit eventが必ず残るか

---

# 7. Kanadeリポジトリ向けの具体的レビュー

Rust製の長時間稼働daemon・control planeでは、通常のlint以外に状態遷移と権限境界が重要です。

## 常時実行

```text
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test / cargo nextest
cargo audit
cargo deny
```

## リスクに応じた追加実行

- `unsafe`変更：Miri
- 並行処理・lock・channel変更：Loomまたはモデル化テスト
- parser・protocol変更：property-based test / fuzzing
- event schema変更：旧イベントのreplay test
- provider adapter変更：contract test
- retry・timeout変更：fault injection
- daemon再起動処理：crash/recovery test

## Kanade固有のArchitecture Invariant

Botは次の不変条件をコードグラフとテストの両方で確認すべきです。

```text
Provider Adapterはpermission gateを迂回できない
Provider Adapterはbudget gateを迂回できない
承認必須操作はhuman approvalなしにcommitされない
Event/Audit logはappend-only
同じeventの再処理はidempotent
cancel後に結果がcanonical stateへ反映されない
retryしても二重課金・二重書き込みが起きない
capability manifest外のtoolを実行できない
Shioriの正本はKanadeから無条件に上書きされない
```

これらは一般的な「バグがないか」というプロンプトでは見つかりにくいため、明示的なルール・状態遷移テストとして保持すべきです。

---

# 8. 評価方法

## 8.1 自分のリポジトリでReplay Benchmarkを作る

サービスを比較する際は、公開ベンチマークだけでなく、過去のPRを固定して再実行します。

評価データには次を含めます。

- 人間が実際に指摘した問題
- 修正につながったコメント
- 却下されたコメントと理由
- merge後のrevert
- production incident
- 後から見つかった不具合
- 意図的に埋め込んだseeded bug
- 正常な変更で誤検出しやすいケース

人間のレビューコメントだけを完全な正解とすると、人間も見逃した問題を評価できません。障害、revert、テスト、人工的な変異も組み合わせます。

---

## 8.2 測定すべき指標

### 品質

- issue-level precision
- severity-weighted recall
- 正しいファイル・関数・行へのlocalization
- actionable rate
- verified evidence rate
- duplicate rate
- outdated rate
- pre-existing issue誤報率

### 人間への効果

- 修正につながった割合
- 人間の確認時間
- time-to-first-actionable-finding
- PR cycle time
- 人間のコメント数
- レビュー待ち時間

### 長期成果

- merge後の不具合
- revert率
- incident数
- escaped critical defect
- 同じ問題の再発率

### 運用

- PRあたりのLLMコスト
- CI時間
- p50/p95 latency
- sandbox失敗率
- model/provider別の成功率

「レビューコメント数」はKPIにしない方がよいです。

---

## 8.3 初期目標

これは外部ベンチマークではなく、導入時の設計目標です。

```text
blocking finding:
  precision 95%以上
  100%に静的証拠または実行証拠を要求

inline finding:
  precision 80%以上
  原則1 PRあたり3〜5件以下

quality:
  duplicate / outdated 各10%未満
  pre-existing issueをblockingしない
  suggested patchはbuild/test通過後のみ公開
```

高い再現率を狙った候補は内部に保存し、公開側は精度優先にします。

---

# 9. 導入順序

## Phase 1：決定的基盤

- GitHub Appまたは中央workflow
- base/head SHA固定
- branch protection
- required reviews
- merge queue
- CodeQL
- dependency review
- lint、typecheck、test
- GitHub Checkへの結果集約

## Phase 2：LLMのShadow Review

- コメントを公開せず、内部でFindingを生成
- 過去PRと新規PRで精度を測る
- Copilot、Codex、Claude、Qodo、CodeRabbit、Greptileなどを同条件で比較
- モデル別ではなく、問題カテゴリ別に測定する

## Phase 3：高信頼コメントのみ公開

- Judgeと重複排除を導入
- inlineコメント数を制限
- 修正・却下理由を収集
- incremental reviewを実装

## Phase 4：実行検証

- regression test生成
- base/head二重実行
- suggested patch検証
- migration replay
- cross-repository contract check

## Phase 5：限定的なマージゲート

- 決定的違反
- 再現テストで証明された重大問題
- 明確なセキュリティdata flow
- 承認済みArchitecture Invariant違反

だけをblocking対象にします。

---

# 最終的な推奨構成

最短で高品質な構成は、GitHubネイティブ機能を基礎に置くことです。

```text
GitHub branch rules / merge queue
+ CodeQL
+ Dependency Review
+ GitHub Code Quality（利用可能なプランの場合）
+ Copilot Code Review
```

その上で、CodeRabbit、Qodo、Greptileのうち一つをShadow Modeで比較します。複数サービスを同時にPRへコメントさせるのではなく、それぞれのFindingを一つのJudgeへ集約すべきです。

最終形としては、外部サービスをレビューの権威にするのではなく、

```text
Shiori
  = ルール・仕様・評価データ・人間承認の正本

Kanade
  = モデル・解析器・サンドボックス・検証・GitHub連携の実行基盤
```

とし、Copilot、Codex、Claude、CodeRabbit的な解析、Qodo的なJudge、Greptile的なコードグラフ、BitsAI的な二段フィルタ、Echo的な実行検証、GitHubのCodeQL・Checks・merge queueを統合するのが最も強い設計です。

そのシステムの核心は、**「AIがそう思った」ではなく、「このPRがこの経路でこの問題を新たに導入し、このテストまたは静的証拠で再現できる」と示すこと**です。これが、レビューBotを便利なコメント生成器から、信頼できるソフトウェア品質基盤へ引き上げる境界になります。

---

# you asked

message time: 2026-07-27 17:19:01

今回はaws bedrockを使ったclaude codeのみを使います。
なるべく短時間で、質の良いレビューが欲しいです。
今は必要以上に時間がかかっていたり、古い知識を元にAIがレビューすることが問題になっています。

---

# chatgpt response

Claude CodeをBedrock上で使う前提に絞り、遅延の原因を分解したうえで、古い知識を避けつつ短時間で高精度にする構成へ落とし込みます。

# Bedrock上のClaude Codeだけで最適化する方針

今回は、複数モデル・複数サービスによるレビューは不要です。

目標が

- 短時間
- 高い実用精度
- 古い知識による誤指摘を防ぐ

であれば、最も効くのは次の構成です。

```text
軽量な機械処理
    ↓
関連情報だけを収集
    ↓
Claude Codeで1回レビュー
    ↓
重大な候補だけClaude Codeで再検証
    ↓
最大3〜5件を投稿
```

**複数のClaude CodeエージェントにPR全体を何度も読ませる構成は避けるべきです。**
時間がかかる主因になりやすく、同じコードの再探索、重複指摘、過剰な自己検証が発生します。

---

# 1. 現在の問題の原因

## 必要以上に時間がかかる原因

典型的には次の処理が重複しています。

```text
Claudeがリポジトリ構造を探索
↓
差分を読む
↓
関連ファイルを探す
↓
テストを探す
↓
仕様を探す
↓
サブエージェントも同じ探索を行う
↓
最後にもう一度全体を確認する
```

Claude Codeのサブエージェントは独立したコンテキストを持つため、調査結果によるメインコンテキストの汚染を防げます。一方で、各サブエージェントが同じリポジトリを探索すると、呼び出し回数と総処理時間は増えます。Anthropicも、サブエージェントは大量の探索結果をメイン会話から隔離したい場合に有効だと説明しています。したがって、単純なPRレビューで常時複数起動するより、限定的な調査に使うべきです。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents?utm_source=chatgpt.com))

## 古い知識を使う原因

Claudeは、次のような情報を学習済み知識で補ってしまうことがあります。

- ライブラリの現在のAPI
- フレームワークの最新仕様
- 非推奨になった実装方法
- 現在のセキュリティ推奨事項
- プロジェクト内部の最新設計
- 過去に廃止されたルール

対策は「もっと賢いプロンプト」ではありません。

**レビューで利用可能な情報源を明示し、それ以外の知識だけで指摘することを禁止する**必要があります。

---

# 2. 推奨する最小構成

```text
GitHub Actions
│
├─ 1. PR差分とメタデータ取得
├─ 2. 高速な決定的チェック
├─ 3. Context Pack生成
├─ 4. Claude Code Review
├─ 5. 重大候補だけ検証
└─ 6. GitHub Checkへ投稿
```

Claude CodeはBedrockを正式に利用でき、AWS認証、IAM、モデル設定を通じて実行できます。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/amazon-bedrock?utm_source=chatgpt.com))

---

# 3. 最重要改善：Claudeに探索させすぎない

Claudeに最初から次のように頼むのは遅くなります。

```text
リポジトリ全体を調べて、このPRを詳細にレビューしてください。
必要なファイル、仕様、ドキュメント、テストをすべて探索してください。
```

代わりに、Claude Codeを起動する前にスクリプトで`review-context.md`を生成します。

## `review-context.md`の内容

```markdown
# PR metadata

- Base SHA: ...
- Head SHA: ...
- Changed files: 8
- Changed production LOC: 142
- Languages: TypeScript, SQL
- Risk flags:
  - authentication
  - database migration

# Diff summary

...

# Changed symbols

- src/auth/session.ts
  - validateSession()
  - refreshSession()

# Direct dependencies

- validateSession()
  - called by middleware.ts
  - called by api/session/route.ts

# Relevant tests

- tests/auth/session.test.ts
- tests/api/session-route.test.ts

# Deterministic checks

- TypeScript: PASS
- ESLint: PASS
- Unit tests: PASS
- Migration test: FAIL
- Semgrep: 1 finding

# Repository rules

- docs/architecture/authentication.md
- docs/adr/ADR-014-session-storage.md
- CLAUDE.md

# Dependency versions

- next: 16.2.1
- react: 20.0.0
- drizzle-orm: 0.xx.x
```

Claudeはこのパックを起点にし、必要な場合だけ追加ファイルを読みます。

Anthropicも、巨大なログをそのままClaudeに読ませるのではなく、Hookなどで事前処理し、必要部分だけを渡す方法を推奨しています。また、アーキテクチャや重要ディレクトリをSkillとして提供すれば、毎回の探索を減らせます。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/costs?utm_source=chatgpt.com))

---

# 4. 古い知識を防ぐSource-of-Truth方式

レビュー時の情報源を、次の優先順位に固定します。

```text
1. PRのbase/headに存在する実際のコード
2. lockfileに記録された実際のバージョン
3. リポジトリ内の承認済み仕様・ADR
4. CIで取得した対象バージョンの公式ドキュメント
5. Claudeの一般知識
```

重要なのは、5番だけを根拠とした指摘を公開しないことです。

## レビュー指示

```text
ライブラリ、フレームワーク、APIの仕様について、
学習済み知識だけを根拠に指摘してはならない。

現在の依存バージョンをlockfileまたはmanifestから確認すること。

バージョン依存の主張は、以下のいずれかを証拠として示すこと。

- リポジトリ内の型定義
- インストール済みパッケージの実装
- 対象バージョンの公式ドキュメント
- コンパイラ、テスト、静的解析の結果

確認できない場合は、PRコメントにせず
unverifiedとして内部結果に残すこと。
```

---

# 5. 公式ドキュメントを毎回ウェブ検索させない

「最新情報を確認しろ」とだけ指示すると、すべてのPRでウェブ探索を始めて遅くなります。

代わりに、依存ライブラリごとのドキュメントキャッシュを作ります。

```text
.review-knowledge/
├── manifest.json
├── next/
│   └── 16.2.1/
│       ├── routing.md
│       ├── caching.md
│       └── server-actions.md
├── drizzle/
│   └── 0.xx.x/
└── aws-sdk/
    └── 3.xxx.x/
```

`manifest.json`には次を保存します。

```json
{
  "next@16.2.1": {
    "source": "official-docs",
    "retrievedAt": "2026-07-27T00:00:00Z",
    "commitOrVersion": "16.2.1",
    "expiresAfterDays": 14
  }
}
```

## 更新条件

次の場合だけ更新します。

- lockfileのバージョンが変わった
- 既存キャッシュが期限切れ
- PRが未知のAPIを変更した
- Claudeが仕様確認を要求した
- 重大なsecurity findingを検証する必要がある

これにより、

```text
毎PRでネット検索
```

ではなく、

```text
通常はローカルキャッシュ
必要な場合だけ公式情報を更新
```

になります。

MCPを使えば外部データソースをClaude Codeへ接続できますが、ツール定義自体もコンテキストを消費します。レビュー用途では、大量のMCPを常時有効にせず、公式ドキュメント取得専用MCPを検証用サブエージェントだけに与える方が効率的です。Anthropicの公式資料でも、MCPをサブエージェント側だけに定義し、メインセッションのコンテキスト消費を避けられると説明されています。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/mcp?utm_source=chatgpt.com))

---

# 6. Claude Codeは基本1エージェントにする

推奨構成は次です。

## 通常PR

```text
Main Reviewerのみ
```

## 高リスクPR

```text
Main Reviewer
    ↓
Verifier subagent 1つ
```

高リスク条件は次のように限定します。

- 認証・認可
- データ削除
- DB migration
- 支払い
- secret・暗号
- 並行処理
- public API変更
- CI/CD・権限設定
- Claudeが重大度High以上の問題を発見した

次のような構成は避けます。

```text
correctness agent
security agent
performance agent
architecture agent
test agent
judge agent
```

Claude Codeだけでこれを常時実行すると、精度向上よりも待ち時間とコスト増加が目立ちやすくなります。

AnthropicのClaude Code Code Review自体は、複数の専門エージェント、候補の検証、重複排除、重大度順の並べ替えを採用しています。ただし、これはサービス側で最適化された専用構成です。自前のBedrockワークフローでそのまま模倣すると、複数の完全なClaude Codeセッションが走り、かなり重くなり得ます。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/code-review?utm_source=chatgpt.com))

---

# 7. 2段階レビューにする

## Pass 1：高速レビュー

目的は、問題候補の抽出です。

Claudeに許可する処理を絞ります。

```text
Read
Glob
Grep
git diff
git show
限定されたテストコマンド
```

禁止または制限するもの：

```text
無制限なウェブ検索
リポジトリ全体の再帰的読取り
全テスト実行
ビルドの繰り返し
複数サブエージェント
コード修正
```

出力は構造化します。

```json
{
  "findings": [
    {
      "id": "F1",
      "severity": "high",
      "confidence": 0.84,
      "path": "src/auth/session.ts",
      "line": 91,
      "claim": "...",
      "evidence": [
        "src/auth/session.ts:91",
        "src/middleware.ts:44"
      ],
      "requiresVerification": true
    }
  ]
}
```

Claude Codeは非対話実行の`claude -p`とJSON出力に対応しているため、CIからプログラム的に扱えます。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/cli-reference?utm_source=chatgpt.com))

## Pass 2：選択的検証

次のものだけ検証します。

```text
severity >= high
または
confidenceが中程度だが影響が大きい
```

VerifierにはPR全体を再レビューさせません。

```text
Finding F1が正しいかだけを確認してください。

確認事項:
1. PRが新たに導入した問題か
2. 実行経路が存在するか
3. 現在の依存バージョンで成立するか
4. テストまたは型検査で再現できるか
5. 既存コードにより防止されていないか
```

これで、1回目の探索をほぼ再利用できます。

---

# 8. レビュー対象を絞る

## 原則除外するファイル

```text
lockfile
generated code
vendor
snapshots
minified files
画像・バイナリ
自動生成されたmigration metadata
formatterだけの変更
```

ただし、lockfileは依存関係変更の確認には使います。

## 自動でレビューを軽くする条件

```text
docs-only
test-only
rename-only
format-only
generated-only
```

これらは通常の詳細レビューをせず、

```text
決定的チェック
+ 変更要約
```

だけで終了します。

---

# 9. テストをClaudeに何度も実行させない

レビュー開始前にCI側で一度だけ実行します。

```text
typecheck
lint
変更範囲のunit test
静的解析
```

結果をClaudeに渡します。

悪い構成：

```text
CIでテスト
↓
Claudeが同じテストを実行
↓
Verifierがもう一度同じテストを実行
```

良い構成：

```text
CIで一度実行
↓
結果をContext Packへ保存
↓
Claudeは必要な追加テストだけ実行
```

HooksはClaude Codeのライフサイクル各所でシェルコマンドやHTTP処理を実行できるため、ログの要約、許可コマンドの制御、コンテキスト注入などに使えます。([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/hooks?utm_source=chatgpt.com))

---

# 10. 推奨プロンプト

```text
あなたはPRの欠陥検出を行うレビュアーです。

目的:
このPRが新たに導入する、修正価値の高い問題だけを検出してください。

優先順位:
1. correctness
2. security
3. data loss
4. concurrency
5. backward compatibility
6.重大なperformance regression

対象外:
- style
- naming
- formatterで修正できる問題
- 主観的な設計上の好み
- PR以前から存在する問題
- 変更行と因果関係のない問題
- 証拠のない一般論

情報源の優先順位:
1. PRのbase/headのコード
2. lockfileと型定義
3. review-context.md
4. リポジトリ内の仕様、ADR、CLAUDE.md
5. 対象バージョンの公式ドキュメント

禁止事項:
- 学習済み知識だけを根拠に、現在のライブラリ仕様を断定しない
- リポジトリ全体を無条件に探索しない
- 全テストを無条件に実行しない
- 問題数を増やすための推測をしない
- 同じ原因を複数コメントに分割しない

手順:
1. review-context.mdとPR差分を読む
2. 変更された振る舞いを特定する
3. 必要な直接依存だけ読む
4. 問題候補を生成する
5. 既存の防止処理がないか確認する
6. PRが導入した問題かbaseと比較する
7. 証拠付きのFindingをJSONで返す

各Findingには必ず含める:
- severity
- path
- line
- claim
- concrete failure scenario
- evidence
- introduced_by_pr
- confidence
- requires_verification

確信度が低い事項は公開Findingに含めない。
Findingがなければ空配列を返す。
最大5件。
```

---

# 11. モデルの使い分け

短時間で質を求める場合は、通常レビューを**最新のSonnet系**に統一するのが基本です。

```text
通常PR:
  Sonnet
  1回のレビュー
  extended thinkingは最小または無効

高リスクPR:
  Sonnet
  必要なFindingだけ追加検証

極めて重要なPR:
  より強いモデルを検証パスだけに使用
```

Extended thinkingは処理時間を増やすため、全PRで常時有効にするべきではありません。AWS公式資料も、thinking blockの生成により応答時間が増えることを明記しています。([AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-extended-thinking.html?utm_source=chatgpt.com))

モデルIDを`latest`のような曖昧な指定にせず、明示的に管理します。

```text
REVIEW_MODEL_ID=...
REVIEW_MODEL_REVISION=...
REVIEW_PROMPT_VERSION=...
```

ただし、古いモデルの固定運用を防ぐため、定期的に利用可能モデルを確認し、評価データで比較してから更新します。

---

# 12. Bedrock側の高速化

## Prompt caching

Bedrockは、対応するClaudeモデルに対してprompt cachingを提供しており、同じ大きなプロンプトprefixの再計算を減らして、レイテンシーと入力コストを下げられます。([AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html?utm_source=chatgpt.com))

キャッシュしやすい順序にします。

```text
固定部分:
  system prompt
  レビュー規則
  Finding JSON schema
  ツール定義
  リポジトリ共通アーキテクチャ

可変部分:
  PRメタデータ
  diff
  CI結果
  関連ファイル
```

つまり、プロンプトの先頭を毎回同一に保ちます。

ただし、Claude CodeがBedrockへの各リクエストでどこまでキャッシュを自動活用するかは、使用バージョンとBedrock統合方式を実測すべきです。CloudWatchまたはBedrockの利用ログでcache read/write相当の指標を確認し、効果が出ていなければ、プロンプト順序とClaude Codeのバージョンを見直します。

## 同一リージョンを使う

GitHub Actions RunnerまたはセルフホストRunnerとBedrockの利用リージョンを近づけます。

```text
日本でセルフホスト:
  ap-northeast-1を第一候補

GitHub-hosted runner:
  実測して最も安定するリージョンを選択
```

クロスリージョン推論を使う場合も、可用性と速度を実測して決めます。BedrockはAnthropic Claudeを含むモデル向けにinference profileを提供しています。([AWS Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html?utm_source=chatgpt.com))

---

# 13. 時間制限を明示する

Claude Codeに無制限に調査させないことが重要です。

推奨する初期値は次です。

```text
通常PR:
  Claude実行 1回
  最大Finding 5件
  サブエージェント 0
  追加テスト 最大1個
  公式ドキュメント取得 最大2ページ

高リスクPR:
  Claude実行 最大2回
  Verifier 最大1
  検証対象Finding 最大3件
  追加テスト 最大2個
```

時間切れの場合は、低品質なコメントを無理に作らせず、

```text
review_incomplete
```

としてCheck summaryに残します。

---

# 14. コメント公開基準

以下を満たすものだけinline commentにします。

```text
introduced_by_pr = true
AND exact_location exists
AND concrete_failure_scenario exists
AND evidence count >= 1
AND confidence >= 0.80
```

マージを止める条件は、さらに厳しくします。

```text
severity in [critical, high]
AND confidence >= 0.95
AND (
  failing_test exists
  OR compiler/static-analysis evidence exists
  OR direct deterministic proof exists
)
```

モデル自身が「重大」と言っただけではブロックしません。

---

# 15. 最終推奨構成

今回の条件では、次が最もバランスが良いです。

```text
┌─────────────────────────────┐
│ GitHub PR                    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Fast precheck               │
│ diff分類・typecheck・test    │
│ 依存バージョン確認           │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Context Pack generator      │
│ 関連ファイル・仕様・CI結果   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Claude Code / Bedrock       │
│ Sonnet・単一レビューパス     │
│ 最大5 findings              │
└──────────────┬──────────────┘
               ↓
        High findingあり？
          ├─ No → 公開
          └─ Yes
               ↓
┌─────────────────────────────┐
│ Claude Code verifier        │
│ Findingだけを限定検証       │
│ 必要時のみ公式docsを参照     │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Deduplicate & publish       │
│ 最大3〜5コメント            │
└─────────────────────────────┘
```

## やめるべきこと

- PRごとに複数専門エージェントを常時起動する
- リポジトリ全体を毎回読み直す
- 同じテストを複数エージェントが実行する
- 常にウェブ検索する
- Extended thinkingを全レビューで使う
- 学習済み知識だけでAPIの廃止・変更を断定する
- styleや軽微な改善までコメントする
- 全候補を別セッションで再レビューする

## 導入すべきこと

- `review-context.md`の事前生成
- lockfileに基づくバージョン確認
- バージョン別の公式ドキュメントキャッシュ
- Sonnetによる単一レビュー
- High以上だけ選択的に再検証
- 最大コメント数の制限
- JSON Schemaによる構造化出力
- Findingごとの根拠と情報源
- PR以前から存在する問題の除外
- 実行時間、ツール回数、探索範囲の上限

特に大きな改善が見込めるのは、**サブエージェントを増やすことではなく、Claude Codeの前段で適切なContext Packを作ること**です。これにより探索時間を削減しつつ、コード、実際の依存バージョン、最新の公式仕様に基づくレビューへ変えられます。

---

# you asked

message time: 2026-07-27 17:33:08

Context packの中に入れれる情報について調査したいです。より良いレビューのためにはPRに関連するdiffだけでなく、リポジトリ全体についてよくわかっているエージェントもレビューに参加することが望ましいですが、その文脈を与えることはできるのですか？

また、正しい情報に基づいて応えさせるために、ライブラリの最新仕様について知るためのcontext7 deepwiki mcpのようなツールを持たせるのもありだと考えていますが、何がいいアイディアはありますか？

---

# chatgpt response

Context Packを「PR差分用」と「リポジトリ理解用」に分け、さらに外部知識取得を必要時だけ呼ぶ構成として整理します。Claude Codeの公式機能、Context7、DeepWiki、MCP運用上の注意点まで比較します。

# 結論

**リポジトリ全体を理解したエージェントをレビューに参加させることは可能です。**
ただし、リポジトリ全体を毎回Context Packへ詰め込むのではなく、次の三層に分けるのが適切です。

```text
常時ロードする小さな知識
    ├─ アーキテクチャ概要
    ├─ 重要な不変条件
    ├─ ディレクトリ責務
    └─ レビュー規則

必要時に検索するリポジトリ知識
    ├─ 関連するコード
    ├─ caller / callee
    ├─ ADR・仕様
    ├─ 過去PR・issue
    └─ Git履歴

必要時に取得する外部知識
    ├─ Context7
    ├─ 公式ドキュメント
    ├─ ライブラリ本体の型・ソース
    └─ DeepWiki
```

そしてレビューを、

```text
PR Reviewer
    +
Repository Expert
    +
External Knowledge Verifier
```

という役割に分けます。

重要なのは、これらを全員が最初から大量の文脈を読む並列エージェントにしないことです。
**Repository Expertは常設の知識基盤ではあるが、PRごとに必要な質問へだけ答える検索型エージェント**にするのが、速度と精度の両立に最も向いています。

---

# 1. Context Packに入れられる情報

Context Packは、単なるdiffの寄せ集めではなく、次のような構造にできます。

## A. PRそのものの情報

```text
PRタイトル・本文
関連issue
base SHA / head SHA
変更ファイル
変更されたシンボル
diff
追加・削除・変更の分類
作者が意図した振る舞い
変更された依存関係
CI結果
```

PR本文はそのまま信用するのではなく、**「作者の意図」**として扱います。

---

## B. 直接影響を受けるコード

変更された行だけでなく、以下を含めます。

```text
変更された関数・クラスの完全な定義
変更された型・interface
直接のcaller
直接のcallee
import元・import先
変更されたAPIの利用箇所
関連テスト
関連する設定
関連するDB schema・migration
```

たとえば、次のdiffだけでは不十分です。

```ts
return authorize(request.userId, resource);
```

必要な文脈は次です。

```text
request.userIdはどこから来るか
authorize()は何を信頼しているか
middlewareで認証済みか
同じresourceへの別経路はあるか
既存テストでは何を保証しているか
```

---

## C. リポジトリ全体の構造

Repository Expertに必要なのは、全文ではなく**圧縮された構造モデル**です。

```text
リポジトリの目的
主要コンポーネント
各ディレクトリの責務
依存方向
主要なデータフロー
主要な状態遷移
public API
永続化方式
認証・認可境界
外部サービスとの境界
重要なバックグラウンド処理
```

例：

```yaml
repository:
  purpose: >
    AIエージェントの実行、権限管理、監査記録を担当するdaemon

components:
  scheduler:
    path: crates/scheduler
    responsibility: WorkflowRunの実行順序を決定する
    may_depend_on:
      - domain
      - event_store
    must_not_depend_on:
      - provider_specific_code

  provider_adapters:
    path: crates/providers
    responsibility: 外部AIプロバイダとの通信
    invariants:
      - permission gateを迂回してはならない
      - budget gateを迂回してはならない
```

この情報は、リポジトリ全体の「地図」です。

---

## D. アーキテクチャ上の不変条件

通常のドキュメントより重要です。

```text
認可は必ずsession subjectを使う
Provider Adapterは直接DBへ書かない
承認前の成果物をcanonical stateへ移さない
イベント処理は冪等である
migrationは旧バージョンから適用できる
公開APIの削除には互換期間を設ける
```

これらを構造化しておくと、Repository ExpertがPRを見て、

```text
この変更はどの不変条件に関係するか
```

を返せます。

---

## E. リポジトリ固有の用語と概念

Claudeは一般的な意味で用語を解釈しがちです。

たとえばKanadeでは、

```text
Task
WorkflowRun
WorkerInvocation
ClaimLedger
EvidenceBundle
Canonical State
```

の意味が一般的なソフトウェア用語と異なる可能性があります。

そこで、Context PackにはGlossaryを含めます。

```yaml
terms:
  EvidenceBundle:
    meaning: >
      エージェントの判断を支えるコード、テスト、ログ、
      外部資料などの証拠集合
    not_equivalent_to:
      - plain log
      - final response
```

---

## F. ADR・仕様・設計判断

次のような資料を関連度に応じて取得します。

```text
Architecture Decision Records
設計書
API仕様
データモデル仕様
セキュリティモデル
運用ルール
過去の廃止判断
```

全文を入れず、

```text
決定
理由
適用範囲
置き換えた旧設計
現在も有効か
関連ファイル
```

だけに圧縮します。

---

## G. 過去のPR・issue・レビュー判断

これは非常に価値があります。

```text
同じファイルを変更した過去PR
同じAPIに関する過去のバグ
過去に却下された設計
過去レビューで繰り返し指摘された問題
障害やrevertにつながった変更
```

たとえば、

```text
以前、このキャッシュは並行更新時に壊れた
```

という情報は、コードだけを見ても分かりません。

ただし、過去PRを大量に渡すのではなく、次のようにFinding単位へ変換します。

```yaml
historical_lessons:
  - id: HIST-014
    affected_symbols:
      - SessionCache.refresh
    lesson: >
      refreshとinvalidateが並行すると古いsessionが復活する可能性がある
    evidence:
      - PR-842
      - incident-2026-04-18
    status: active
```

---

## H. 実行時の知識

可能であれば以下も有用です。

```text
テスト結果
型エラー
lint結果
coverage差分
実行ログ
benchmark
production error
feature flag
現在の設定値
```

ただしproduction情報をClaudeへ渡す場合は、秘密情報・個人情報の除去が必要です。

---

# 2. リポジトリ全体を理解したエージェントは作れるか

作れます。ただし「毎回リポジトリ全体を読むエージェント」ではなく、**Repository Expert Agent**として設計します。

Claude Codeのカスタムサブエージェントは、独立したコンテキスト、専用のsystem prompt、利用可能なツール、権限を持てます。大量の探索結果をメインレビューのコンテキストへ流さず、要約だけを返す用途に適しています。([Claude](https://code.claude.com/docs/en/sub-agents?utm_source=chatgpt.com))

## 推奨する役割分担

### Main PR Reviewer

持つ情報：

```text
PR差分
直接関連コード
CI結果
Repository Expertの回答
外部仕様の検証結果
```

役割：

```text
PRが導入する問題を判断する
Findingを統合する
コメントを書く
```

### Repository Expert

持つ情報：

```text
リポジトリ構造
コード検索
シンボルグラフ
ADR
仕様
過去PR・issue
Git履歴
不変条件
```

役割：

```text
変更の影響範囲を特定
関連する不変条件を返す
類似実装を探す
過去の設計判断を返す
既存の防止機構を確認する
```

### External Knowledge Verifier

持つツール：

```text
Context7
公式ドキュメント取得
ライブラリソース検索
必要ならDeepWiki
```

役割：

```text
バージョン依存の主張を検証
非推奨APIか確認
公式の推奨方法を確認
```

---

# 3. Repository Expertに渡す文脈

## 方法1：CLAUDE.md

Claude Codeはセッション開始時に`CLAUDE.md`とauto memoryを読み込みます。ただし、これらは強制設定ではなく「文脈」として扱われるため、絶対に守らせる制約はHooksなどで実装すべきです。また、指示は短く具体的な方が安定します。([Claude](https://code.claude.com/docs/en/memory?utm_source=chatgpt.com))

`CLAUDE.md`には次だけを置きます。

```markdown
# Repository purpose

Kanade is the execution and orchestration plane for AI agents.

# Critical invariants

- Provider adapters must not bypass permission checks.
- Provider adapters must not bypass budget checks.
- Approval-required operations must not alter canonical state before approval.
- Event processing must be idempotent.
- Every state transition must produce an audit event.

# Architecture index

- Domain model: docs/architecture/domain.md
- Permissions: docs/architecture/permissions.md
- Event model: docs/architecture/events.md
- Provider adapters: docs/architecture/providers.md

# Review behavior

- Treat base-branch specifications as authoritative.
- Do not report pre-existing issues as PR findings.
- Verify version-dependent claims from installed source or official documentation.
```

**詳細な設計書全文は置かない方がよいです。**

---

## 方法2：`.claude/rules/`

パス別ルールを分離します。

```text
.claude/rules/
├── database.md
├── provider-adapters.md
├── event-store.md
├── security.md
└── frontend.md
```

たとえば`provider-adapters.md`：

```markdown
---
paths:
  - "crates/providers/**"
---

- Provider code must call the permission gate before execution.
- Provider code may not directly mutate canonical state.
- Retry behavior must not duplicate billing or side effects.
- Provider-specific errors must be converted to domain errors.
```

これにより、無関係なPRに全ルールを持ち込むのを避けられます。Claude Codeはプロジェクトの`.claude`ディレクトリからinstructions、rules、skills、subagentsなどを読み込めます。([Claude](https://code.claude.com/docs/en/claude-directory?utm_source=chatgpt.com))

---

## 方法3：Repository Knowledge Index

これが最も重要です。

リポジトリのbase branchから定期的に、次の成果物を生成します。

```text
.review-index/
├── repository-summary.json
├── components.json
├── symbols.jsonl
├── relationships.jsonl
├── invariants.yaml
├── adr-index.jsonl
├── glossary.yaml
├── test-map.jsonl
├── history-lessons.jsonl
└── freshness.json
```

### `components.json`

```json
{
  "components": [
    {
      "name": "event_store",
      "paths": ["crates/event-store/**"],
      "responsibilities": [
        "persist domain events",
        "replay aggregate state"
      ],
      "depends_on": ["domain"],
      "used_by": ["scheduler", "api"],
      "invariants": ["EVENT-001", "EVENT-002"]
    }
  ]
}
```

### `relationships.jsonl`

```json
{"from":"ApiHandler.startRun","relation":"calls","to":"Scheduler.start"}
{"from":"Scheduler.start","relation":"writes","to":"EventStore.append"}
{"from":"ProviderAdapter.execute","relation":"requires","to":"PermissionGate.check"}
```

### `test-map.jsonl`

```json
{
  "symbol": "Scheduler.start",
  "tests": [
    "crates/scheduler/tests/start_run.rs",
    "tests/integration/workflow_execution.rs"
  ]
}
```

Repository Expertはこれを最初に検索し、必要なコードだけを読みます。

---

# 4. インデックスをどう作るか

最初から高度な知識グラフを作る必要はありません。

## 第1段階：機械的に生成

```text
ファイルツリー
language
package/module
import関係
関数・クラス・型
参照箇所
テストとの対応
Git履歴
CODEOWNERS
```

使用候補：

```text
tree-sitter
LSP
SCIP
ctags
ripgrep
git log
git blame
言語固有コンパイラ
```

## 第2段階：Claudeに要約させる

base branch更新時だけ、Claude Codeに次を生成させます。

```text
コンポーネント責務
主要データフロー
アーキテクチャ境界
重要な不変条件候補
用語集
```

ただし、Claude生成物をそのまま正本にしません。

```text
generated
    ↓
人間または既存設計書で確認
    ↓
approved
```

という状態を持たせます。

## 第3段階：増分更新

PRごとに全インデックスを作り直さず、

```text
base SHAのインデックス
+
変更シンボルだけ増分更新
```

します。

これにより速度を維持できます。

---

# 5. Repository Expertの実際の問い合わせ

Main Reviewerが直接リポジトリ全体を探索する代わりに、次の質問をします。

```json
{
  "changed_symbols": [
    "SessionService.refresh",
    "SessionRepository.update"
  ],
  "questions": [
    "この変更が破る可能性のあるアーキテクチャ不変条件は何か",
    "直接・間接的な利用箇所はどこか",
    "類似処理はどこにあるか",
    "関連するADRと過去の問題は何か",
    "既存テストが保証していない経路は何か"
  ]
}
```

Repository Expertの回答：

```json
{
  "affected_components": [
    "authentication",
    "api-middleware"
  ],
  "invariants": [
    {
      "id": "AUTH-003",
      "summary": "refresh token rotation must be atomic",
      "source": "docs/architecture/auth.md"
    }
  ],
  "callers": [
    "middleware.refreshExpiredSession",
    "api.session.refresh"
  ],
  "similar_implementations": [
    "ApiKeyService.rotate"
  ],
  "historical_lessons": [
    {
      "id": "HIST-014",
      "summary": "concurrent refresh previously restored a revoked token"
    }
  ],
  "recommended_files": [
    "src/auth/session-service.ts",
    "src/middleware.ts",
    "tests/auth/concurrent-refresh.test.ts"
  ]
}
```

Main Reviewerにはこの回答と推奨ファイルだけを渡します。

---

# 6. 「全体を知っているエージェント」を永続化できるか

完全な意味で、Claudeのモデル内部にリポジトリ全体を学習させて永続化するわけではありません。

実現するのは次の組み合わせです。

```text
固定知識：
  CLAUDE.md
  .claude/rules
  approved architecture docs

検索可能な知識：
  Repository Knowledge Index
  Git history
  ADR
  issue / PR history

セッション内の知識：
  Repository Expertの独立context

蓄積される補助知識：
  Claude Code auto memory
```

Claude Codeのauto memoryはリポジトリ単位で保存されますが、先頭部分だけがセッション開始時に読み込まれ、Claude自身が書く補助的な記憶です。正確性が必要な設計仕様の正本には向きません。([Claude](https://code.claude.com/docs/en/glossary?utm_source=chatgpt.com))

したがって、

```text
auto memory = 作業上の補助
approved repository index = 正本
```

と分けるべきです。

---

# 7. Context7は有効か

**かなり有効です。**

Context7は、ライブラリの現在またはバージョン固有のドキュメントとコード例を、MCP経由で提供することを目的としています。Claude Code向けの接続方法も公式に提供されています。([Context7](https://context7.com/docs/overview?utm_source=chatgpt.com))

特に次のレビューに向きます。

```text
ライブラリAPIの使い方
現在の設定項目
非推奨API
バージョン固有の移行方法
フレームワークの推奨パターン
```

ただし、Context7を毎回無条件に呼ぶのは適切ではありません。

## Context7を呼ぶ条件

```text
package manifestまたはlockfileが変更された
新しい外部APIを使用している
Reviewerがバージョン依存の疑いを持った
非推奨・削除・互換性について指摘しようとしている
フレームワーク設定が変更された
セキュリティ関連APIが変更された
```

## Context7を呼ばないケース

```text
単純な内部ロジックのバグ
リポジトリ固有の設計違反
明らかなnull処理
既存テストで再現できる問題
コンパイラが既に証明している問題
```

---

# 8. Context7の検索にはlockfile情報を必ず付ける

悪い問い合わせ：

```text
Next.jsでこのAPIは正しいですか？
```

良い問い合わせ：

```text
package:
  next

installed_version:
  16.2.1

api:
  revalidatePath

question:
  このバージョンでServer Action外から呼び出せるか。
  非推奨または挙動変更があるか。
```

依存バージョンの取得優先順位は次です。

```text
1. lockfile
2. package manifest
3. インストール済みpackage metadata
4. Context7
```

Context7の回答にも、次の情報を要求します。

```json
{
  "library": "next",
  "version_requested": "16.2.1",
  "version_resolved": "16.2.1",
  "claim": "...",
  "source_section": "...",
  "code_example": "...",
  "version_match": "exact"
}
```

`version_match`が`latest`や`approximate`の場合は、重大なレビューコメントの根拠に単独で使わない方が安全です。

---

# 9. DeepWikiは有効か

DeepWikiは、公開GitHubリポジトリについて、アーキテクチャ説明、ソースリンク、コードベースへのQ&Aを提供します。公式のDeepWiki MCPは、公開リポジトリのドキュメントと検索機能へのアクセスを提供し、認証不要で利用できます。([Devin Docs](https://docs.devin.ai/ja/work-with-devin/deepwiki-mcp?utm_source=chatgpt.com))

## DeepWikiが向いているもの

```text
外部ライブラリの内部構造を理解する
OSSの実装方式を調べる
公式ドキュメントだけでは分からないコード経路を見る
依存先のアーキテクチャを把握する
```

例：

```text
このライブラリは内部でリトライするのか
このAPIはどの状態を変更するのか
この設定はどのコード経路で参照されるか
```

## DeepWikiが向いていないもの

```text
あなたの非公開リポジトリの正本
正確なバージョン固有API仕様の唯一の根拠
セキュリティ判断の唯一の根拠
最新リリースの厳密な互換性判定
```

公開DeepWikiとDeepWiki MCPが提供するのは基本的なドキュメント・Q&A機能であり、高度なコード検索などの完全な機能はDevin側にあります。([Devin Docs](https://docs.devin.ai/work-with-devin/deepwiki?utm_source=chatgpt.com))

したがって、レビュー用途の優先順位は、

```text
インストール済み型・ソース
    ↓
Context7または公式ドキュメント
    ↓
DeepWikiで実装構造を補完
```

がよいです。

---

# 10. Context7とDeepWikiの使い分け

| 質問 | 第一候補 |
|---|---|
| 現在のAPI仕様 | Context7 |
| バージョン固有のコード例 | Context7 |
| 非推奨・移行手順 | Context7＋公式文書 |
| OSS内部の実装経路 | DeepWiki |
| OSSのアーキテクチャ | DeepWiki |
| 自分のリポジトリ設計 | Repository Expert |
| 自分の過去PR・issue | GitHubまたは独自インデックス |
| 実際に導入済みの型 | `node_modules`等のローカルファイル |
| 実際の動作 | テスト・再現コード |

---

# 11. さらに有効な情報源

## 11.1 インストール済みライブラリの型定義

TypeScriptなら、

```text
node_modules/package/**/*.d.ts
```

は非常に強い根拠です。

```text
そのバージョンにそのAPIが存在するか
引数型
戻り値
deprecated annotation
overload
```

を確認できます。

一般知識やウェブ情報よりも、**実際にインストールされたバージョンの型**を優先できます。

---

## 11.2 インストール済みソースコード

型だけでは挙動が分からない場合に使います。

```text
node_modules
Cargo registry source
Python site-packages
Go module cache
```

ただし、レビュー時間を増やすため、次の場合だけ読ませます。

```text
重大なFinding候補
公式文書では曖昧
実装依存の挙動
```

---

## 11.3 公式ドキュメント専用MCP

Context7で不足するライブラリ向けに、独自のread-only MCPを用意する方法もあります。

ツールを次の3個程度に絞ります。

```text
resolve_official_docs(package, version)
search_official_docs(package, version, query)
fetch_official_section(document_id, section)
```

返却データには必ず、

```text
取得元ドメイン
取得日時
対象バージョン
文書バージョン
見出し
引用位置
```

を付けます。

一般ウェブ検索を許可するより、対象ドメインを公式サイトだけに限定した方が安全です。

---

## 11.4 GitHub履歴用MCP

自分のリポジトリに関しては、次を検索できると有用です。

```text
関連issue
過去PR
レビューコメント
revert
release note
同じシンボルを変更したcommit
```

ただし、GitHub APIを毎回大量に呼ぶのではなく、PR作成時にbase branchまでの履歴をローカルDBへ同期する方が速くなります。

---

# 12. MCPを増やしすぎても大丈夫か

現在のClaude CodeにはMCP tool searchがあり、ツールの完全なschemaを最初からすべてコンテキストへ入れず、必要時に検索・ロードできます。公式資料では、デフォルトでtool searchが有効になり、必要なツールだけがコンテキストへ入る構成が説明されています。([Claude](https://code.claude.com/docs/en/mcp?utm_source=chatgpt.com))

そのため、以前より複数MCPを接続しやすくなっています。

ただし、実際の問題はコンテキスト量だけではありません。

```text
Claudeがどのツールを使うか迷う
似たツールを重複実行する
外部通信で遅くなる
回答の出典が混ざる
ツール障害でレビュー全体が遅れる
```

したがって、レビューではツールを役割別に制限します。

### Main Reviewer

```text
Read
Grep
Glob
git diff
Repository Expert
```

### Repository Expert

```text
Repository index
Git history
ADR search
code search
```

### External Knowledge Verifier

```text
Context7
DeepWiki
official-docs MCP
local dependency source
```

Main ReviewerにすべてのMCPを直接持たせない方が安定します。

---

# 13. External Knowledge Verifierの起動条件

次の判定をMain Reviewerにさせます。

```json
{
  "claim": "This API was removed in version 16",
  "knowledge_type": "external_versioned",
  "installed_package": "next",
  "installed_version": "16.2.1",
  "verification_required": true
}
```

Verifierはその主張だけを確認します。

```text
PR全体をレビューしない
新しい問題を探索しない
与えられた主張だけを検証する
```

出力：

```json
{
  "verdict": "supported",
  "claim": "revalidatePath is available in Next.js 16.2.1",
  "evidence": [
    {
      "source_type": "installed_type_definition",
      "path": "node_modules/next/cache.d.ts",
      "line": 8
    },
    {
      "source_type": "context7",
      "library": "next",
      "resolved_version": "16.2.1",
      "section": "revalidatePath"
    }
  ],
  "confidence": 0.99
}
```

これを受けて、誤ったFinding候補を削除します。

---

# 14. 情報源の信頼度をスコア化する

外部情報はすべて同じ信頼度ではありません。

```text
1.0  実際のコンパイラ・テスト結果
0.98 インストール済み型定義
0.95 インストール済みソース
0.92 対象バージョンの公式文書
0.88 Context7で完全なバージョン一致
0.80 公式文書のlatest版
0.75 DeepWikiのソース参照付き回答
0.60 過去PRの議論
0.40 Claudeの一般知識
```

Findingを公開する際は、

```text
モデルの確信度
× 情報源の信頼度
× PRへの因果関係
```

で判断します。

Claudeの一般知識だけなら、公開Findingにしません。

---

# 15. 推奨する最終構成

```text
┌────────────────────────────┐
│ Base Branch Knowledge Build │
│ 定期・merge後に増分更新      │
├────────────────────────────┤
│ Architecture map            │
│ Symbol/import/call graph    │
│ ADR/spec index              │
│ Historical lessons          │
│ Test map                    │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│ PR Context Builder          │
├────────────────────────────┤
│ Diff                        │
│ Changed symbols             │
│ Direct dependencies         │
│ CI results                  │
│ Installed versions          │
└──────────────┬─────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
┌──────────────┐  ┌─────────────────┐
│ Main Reviewer│  │Repository Expert│
│ PR中心       │←─│全体構造を検索   │
└──────┬───────┘  └─────────────────┘
       │
       │ 外部仕様に依存する主張だけ
       ▼
┌────────────────────────────┐
│ External Knowledge Verifier│
├────────────────────────────┤
│ local types/source          │
│ Context7                    │
│ official docs MCP           │
│ DeepWiki                    │
└──────────────┬─────────────┘
               ▼
┌────────────────────────────┐
│ Final Finding Filter        │
│ 最大3〜5件                  │
└────────────────────────────┘
```

---

# 私ならこう実装します

## 常時ロード

```text
CLAUDE.md
.claude/rules/
repository-summary.json
invariants.yaml
glossary.yaml
```

合計は短く保ちます。

## Repository Expertがオンデマンド検索

```text
symbols.jsonl
relationships.jsonl
adr-index.jsonl
history-lessons.jsonl
test-map.jsonl
実際のコード
git log / blame
```

## 外部仕様確認

優先順位：

```text
1. lockfileでバージョン確定
2. インストール済み型
3. インストール済みソース
4. Context7
5. 公式ドキュメントMCP
6. DeepWiki
```

## 起動回数

```text
通常PR:
  Main Reviewer
  Repository Expertへの問い合わせ1回
  外部Verifierは必要時だけ

高リスクPR:
  Main Reviewer
  Repository Expert
  Finding単位のExternal Verifier
```

この構成なら、**リポジトリ全体への理解をレビューへ反映しながら、毎回リポジトリ全体をコンテキストへ読み込む無駄を避けられます。**

Context7とDeepWikiの導入も有効ですが、役割は明確に分けるべきです。

```text
Repository Expert
  → 自分のコードベースについて答える

Context7
  → ライブラリのバージョン固有仕様を答える

DeepWiki
  → 外部OSSの内部構造を答える

テスト・型・実装
  → 最終的な事実を証明する
```

これが、速度、コンテキスト効率、情報の新しさ、レビュー精度のバランスが最もよい設計です。

---
Powered by [AI Exporter](https://saveai.net)