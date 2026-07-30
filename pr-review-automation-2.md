> From: https://chatgpt.com/c/6a671966-b524-83ee-8774-2fe8999ba291

# you asked

message time: 2026-07-27 17:42:00

現在、AI Agentの台頭によってgithubのPRの頻度が爆増しており、人間のレビューが追いつかない状況が至る所で見られています。

その問題に対する様々な取り組みについて、最先端のものを広く調査して紹介してください。

---

# chatgpt response

I’ll assess research, OSS, commercial tools, and organizational practices across workload reduction, review quality, risk triage, and AI-change governance, emphasizing maturity, limitations, and effective combinations.

---

# chatgpt response

The frontier is shifting beyond single-LLM diff comments toward specialized agents using repository-wide context, runtime reproduction, and independent verification before findings reach humans through risk-based escalation.

---

# chatgpt response

# 結論

2026年7月27日時点で、最先端の解決策は**「人間の代わりにAIレビュアーを1体置く」ことではありません**。それだけでは、AIが作る大量のPRに対して、今度はAIが大量のレビューコメントを生成し、別のレビュー渋滞を作ってしまいます。

現在進んでいるのは、コード生成から本番投入までを一つの**検証パイプライン**として再設計する動きです。

```text
Issue・仕様
   ↓  PR流入制御、優先度・リスク判定
実装Agent
   ↓  ローカルテスト、静的解析、自己レビュー
証拠付きDraft PR
   ↓
決定論的ゲート
   ↓
複数の専門レビューAgent
   ↓
検証・重複除去・重要度判定
   ↓
リスクに応じて人間へ振り分け
   ↓
Merge Queue
   ↓
Canary・監視・自動Rollback
```

要するに、**人間がすべてのPRを読む体制から、人間の注意を高リスクな判断だけに配分する体制への移行**です。

公開情報だけでは「GitHub全体のPR数がAI Agentによって何％増えたか」を厳密には確定できません。ただしGitHubは、Copilot coding agentの最初の約5か月で100万件を超えるPRがマージされたと報告しています。また、2026年3月時点でCopilot code reviewはGitHub上のコードレビューの5件に1件を超え、1万2,000以上の組織が全PRに自動レビューを実行しているとしています。レビュー能力のスケーリングが現実の課題になっていることは明らかです。 ([The GitHub Blog](https://github.blog/news-insights/octoverse/the-new-identity-of-a-developer-what-changes-and-what-doesnt-in-the-ai-era/))

---

# 1. PRを作る前に減らす「Admission Control」

最も重要な変化の一つは、**Agentが作業を始めた瞬間にPRを開かせない**ことです。

従来のAgent運用では、

1. IssueをAgentに渡す
2. AgentがすぐDraft PRを作る
3. 人間のPR一覧が大量の未完成PRで埋まる

という構造になりがちでした。

現在は、Agentをまず隔離されたbranchやworktree上で動かし、次の条件を満たしたものだけをPRとして昇格させる設計が推奨されています。

- Issueと受け入れ条件が明確
- 実装計画が作成済み
- formatter、lint、型検査、テストが成功
- PR説明と実際のdiffが一致
- 変更が大きすぎる場合は分割済み
- 重複する既存PRがない
- リスク情報とRollback方法が記載済み

GitHub自身も、CopilotにすぐPRを開かせるのではなく、先にリポジトリを調査させ、実装計画を作成し、branch上で変更を反復してからPRを開く方法を案内しています。 ([GitHub Docs](https://docs.github.com/copilot/how-tos/agents/copilot-coding-agent/best-practices-for-using-copilot-to-work-on-tasks))

さらに2026年7月、GitHubはIssue automationに対し、Agentの各操作に「理由」「信頼度」「人間承認」を付ける仕組みを公開しました。高信頼の操作だけ自動適用し、中・低信頼のものを人間に回す設計です。現時点ではIssue操作向けですが、PRレビューでも同じ**confidence-gated automation**が重要になります。ただしGitHubも、この承認機構は利便機能であって、セキュリティ境界ではないと明記しています。 ([The GitHub Blog](https://github.blog/changelog/2026-07-23-agent-automation-controls-in-github-issues-in-public-preview/))

## Shift-left review

PR提出前にAIレビューを実行する動きも広がっています。

- Cursor Bugbotはpush前のローカルレビューを行い、同じdiffをGitHub側で重複レビューしないようにできます。
- CodeRabbit CLIは未コミットの変更をレビューし、修正をローカルで適用できます。
- Claude Code Reviewはローカルの`/code-review`と修正フローを持ちます。

つまり、軽微な問題は共有レビューキューに入る前に処理し、**人間が初めて見る時点で、すでに一度以上レビューされた状態にする**わけです。 ([Cursor](https://cursor.com/blog/bugbot-updates-june-2026))

---

# 2. 「リポジトリ全体を渡す」から「必要な証拠だけ取りに行く」へ

現在のAIレビュー製品は、単純なdiffレビューから、リポジトリ全体、過去のPR、Issue、設計文書、依存関係まで参照するAgent型レビューへ進化しています。

GitHub Copilot code reviewは、必要なリポジトリ文脈を検索し、長いPRではレビュー計画を作り、リンクされたIssueや過去のPRを読み、レビュー間の記憶を利用する設計になっています。 ([The GitHub Blog](https://github.blog/ai-and-ml/github-copilot/60-million-copilot-code-reviews-and-counting/))

しかし、ここには非常に重要な注意点があります。

## コンテキストは多いほどよいわけではない

2026年のSWE-PRBenchでは、8つの先端モデルがdiffだけを与えられた条件でも、人間が指摘した問題の15〜31％しか検出できませんでした。さらに、ファイル内容、AST、import graph、テスト情報などを追加してコンテキストを増やすと、8モデルすべてで成績が低下しました。2,000 token程度の「diff＋構造化された要約」が、2,500 tokenのより豊富な全文脈より良い結果を出しています。 ([arXiv](https://arxiv.org/html/2603.26130v1))

GitHubの内部検証でも、Agentに一般的なリポジトリ探索ツールを与えたところ、

- 広く検索する
- 関係の薄いファイルを読む
- さらに検索対象を広げる
- 不要な文脈を保持する

という探索ループに陥り、コストが増え、検出される問題が減りました。

GitHubはレビュー手順を次のように変更しています。

1. diffから具体的な疑問を作る
2. `grep`や`glob`で候補を絞る
3. 必要なファイルと行範囲だけ読む
4. 証拠が揃った時点で判定する

この変更で、品質を維持しながら平均レビューコストを約20％削減したと報告しています。 ([The GitHub Blog](https://github.blog/ai-and-ml/github-copilot/better-tools-made-copilot-code-review-worse-heres-how-we-actually-improved-it/))

したがって最先端の設計は、

> リポジトリ全体を索引化しておくが、モデルには全部渡さない。
> diffから生まれた具体的な疑問に対して、必要な証拠だけを動的取得する。

というものです。

理想的なContext Packは巨大な固定プロンプトではなく、次の情報を検索できる基盤です。

- 変更されたsymbolの定義と呼び出し元
- 同じパターンを使う既存コード
- 関係するテストとfixture
- linked Issueと受け入れ条件
- ADR、設計書、API仕様
- CODEOWNERSと担当領域
- dependency graph
- lockfileで確定したライブラリのバージョン
- 過去に同種の問題を指摘したPR
- 他リポジトリの利用側コード

Context7、DeepWiki、MCP、社内文書検索なども、常時大量投入するのではなく、**変更されたimport、API、バージョンから必要な情報だけ検索する**方が適しています。

---

# 3. 決定論的ツールとAIレビューを重ねる

AI Agentが台頭しても、コンパイラ、型検査、テスト、CodeQL、Semgrep、secret scan、dependency scanなどの価値は下がりません。むしろ、AIコードが増えるほど重要になります。

役割分担は次のようになります。

| レイヤー | 主な役割 | Merge阻止 |
|---|---|---|
| Formatter・lint | 構文、スタイル、単純規則 | 可 |
| 型検査・ビルド | 型、参照、コンパイル | 可 |
| Unit・integration test | 既知の期待動作 | 可 |
| CodeQL・Semgrep等 | 既知の脆弱性・データフロー | 可 |
| AIレビュー | 意図の不一致、設計、境界条件、未テスト動作 | 原則は助言 |
| 人間 | 要件、トレードオフ、アーキテクチャ、責任判断 | 高リスクで必須 |

GitHub Code Qualityも、決定論的なCodeQLと、AIによる保守性・信頼性の指摘、Copilot Autofixを組み合わせる方向へ進んでいます。GitHubのAI security detectionsは、既存解析を補完しますが、AI生成であることを明示し、informational・non-blockingとして扱われます。 ([The GitHub Blog](https://github.blog/changelog/2026-07-20-github-code-quality-is-now-generally-available/))

GitLabも、AIレビュアーへのcustom instructionsは「ガイダンス」であって強制ポリシーではなく、セキュリティやコンプライアンスの強制には使用しないよう警告しています。 ([GitLab Docs](https://docs.gitlab.com/user/duo_agent_platform/customize/review_instructions/))

したがって、現時点で安全なのは次の方針です。

- Merge blockerは、再現可能なテスト、静的解析、ルール、明確に校正された高信頼判定に限定する
- 一般的なLLMコメントは、原則としてadvisoryにする
- AIが「問題なし」と言ったことを、品質の証明として扱わない
- AIが指摘した問題は、テストや解析で再現できる形に変換する

---

# 4. 全PRを同じようにレビューせず、リスクで振り分ける

PR量が増えた環境では、レビューシステムは単なるバグ検出器ではなく、**人間の注意を配分するスケジューラ**になります。

リスクスコアには、少なくとも次を含めます。

- 認証、権限、決済、個人情報、データ削除に触れるか
- DB schemaやmigrationを含むか
- 公開API、イベントschema、設定形式を変えるか
- 影響する呼び出し元や利用リポジトリの数
- Rollback可能か
- テストの有無とcoverage差分
- 新規依存関係や権限追加
- 既存パターンからの逸脱度
- Agent自身の不確実性
- 人間が書いた変更か、Agentが生成した変更か
- PR説明とdiffに不整合があるか

実務上は、例えば次のように分けられます。

| リスク | 例 | 推奨処理 |
|---|---|---|
| L0 | 文書、format、機械生成物 | CI成功後に自動Merge＋抜き取り監査 |
| L1 | 局所的な内部実装、十分なテストあり | AIレビュー＋人間1名、または成熟後にサンプリング |
| L2 | API、schema、migration、共有ライブラリ、設定 | CODEOWNERと領域専門家 |
| L3 | 認証、決済、個人情報、削除処理、インフラ権限 | 複数承認、security review、段階的deploy |

GitHubのCODEOWNERS、branch protection、rulesets、required reviewsは、このリスクルーティングの強制層として利用できます。 ([GitHub Docs](https://docs.github.com/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners))

重要なのは、**AIのconfidenceだけでリスクを決めないこと**です。パス、依存関係、権限、データ変更、テスト結果などの客観情報を中心にし、AI confidenceは補助信号にします。

---

# 5. 複数Agentによる専門レビューと「Verifier」

現在の先端製品では、1つの巨大なプロンプトで全部を見るより、専門Agentを並列に動かす設計が増えています。

例えば役割を次のように分けます。

- Correctness reviewer
- Security reviewer
- Test adequacy reviewer
- API compatibility reviewer
- Performance reviewer
- Architecture reviewer
- Dependency・license reviewer

Claude Code Reviewは、複数の専門Agentを並列に動かし、その後に候補指摘の検証、重複除去、severity rankingを行う構成を採っています。Qodoも複数Agentとリポジトリ・履歴・仕様情報を組み合わせる方向です。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/code-review))

ただし、単にAgent数を増やすと、コメント量とコストも増えます。したがって最後に独立したVerifierが必要です。

Verifierは各コメントについて、

- 本当にdiffによって発生した問題か
- 具体的なfile、line、call pathがあるか
- 既存テストや仕様と矛盾するか
- 再現方法を示せるか
- 同じ内容の指摘が重複していないか
- blocking、warning、nitのどれか
- 推奨修正によって別の問題を起こさないか

を確認します。

GitHubも「コメント数を増やすこと」を目標にせず、ノイズより沈黙を優先しています。同社公表値では、71％のレビューでactionable feedbackを出し、29％では何もコメントしない設計です。 ([The GitHub Blog](https://github.blog/ai-and-ml/github-copilot/60-million-copilot-code-reviews-and-counting/))

優れた自動レビューは、20個の曖昧な指摘を並べるのではなく、**証拠の強い1〜3個の重要事項を出す**べきです。

---

# 6. Generate → Review → Revise → Re-verify の閉ループ化

一度レビューコメントを生成して終了する方式から、Agentが修正し、再検証する閉ループへ移行しています。

```text
実装Agent
  ↓
Review Agent
  ↓
Fix Agent
  ↓
関連テスト・静的解析を再実行
  ↓
Independent Verifier
```

2026年7月公開のSWE-Review研究では、リポジトリを探索できるAgentによるgenerate-review-reviseループが、固定コンテキストの一回限りのレビューより、PRの合否判定と修正後の問題解決率の両方で優れていました。重要なのは、レビュー文を生成することではなく、**レビューによって実際に修正後の挙動が改善したかを検証すること**です。 ([arXiv](https://arxiv.org/html/2607.06065))

GitHubのAgentic Autofixも、修正を提案した後、元のCodeQLや第三者解析を再実行し、alertが本当に解消したことを確認してからDraft PRを開く構成です。 ([The GitHub Blog](https://github.blog/changelog/2026-07-10-agentic-autofix-for-code-scanning-alerts-in-public-preview/))

ここでの安全原則は次の通りです。

- コードを書いたAgentだけに自己承認させない
- Fix AgentとVerifierの役割を分離する
- 元の問題を検出した解析を必ず再実行する
- full test suiteだけでなく、変更箇所に対応するfocused testを実行する
- 修正後に新しいdiffを再レビューする
- 「コメントがresolvedになった」ことと「問題が解決した」ことを区別する

---

# 7. PRを「説明文」ではなく「証拠パッケージ」にする

今後のPRは単なるdiffではなく、**Evidence Bundleを伴う変更提案**になると考えられます。

最低限、次の情報を機械可読な形で添付します。

```yaml
intent:
  issue: ...
  acceptance_criteria: ...

change:
  affected_components: [...]
  changed_behavior: ...
  preserved_invariants: [...]

verification:
  commands: [...]
  test_results: ...
  coverage_delta: ...
  static_analysis: ...
  security_scan: ...

risk:
  level: L2
  reasons: [...]
  rollout: ...
  rollback: ...

provenance:
  author_type: agent
  model: ...
  agent_config_version: ...
  tools_used: [...]
  external_sources: [...]
  known_uncertainties: [...]
```

Agent生成PRでは、説明文そのものも信頼できるとは限りません。23,247件のAgent生成PRを調べた研究では、コードと説明の重大な不整合があるPRは少数ながら存在し、その中では「実装していない機能を実装したと主張する」phantom claimが大きな割合を占めました。重大な不整合を持つPRは、受理率が低く、Mergeまでの時間も大幅に長くなっています。 ([arXiv](https://arxiv.org/html/2601.04886v2))

したがって、PR descriptionは入力として信用するのではなく、

1. Issueの要求
2. PRの主張
3. 実際のdiff
4. テスト結果

の四者を照合するAgentを置くべきです。

---

# 8. 小さなPR、Stacked PR、Merge Queue

AI Agentは大量のコードを書けるため、一つのPRが巨大化しやすい問題があります。しかし大きなPRほど、

- 人間が全体を理解しにくい
- Agentのコンテキストが散る
- 関係のない変更が混ざる
- 一部の修正で全レビューをやり直す
- CI時間が長くなる
- Merge conflictが増える

という問題が起きます。

GitLabのAIレビューも、大きなMerge Requestではコンテキスト制限によって一部ファイルが解析対象外になる可能性を説明し、変更の分割を推奨しています。 ([GitLab Docs](https://docs.gitlab.com/user/gitlab_duo/code_review/))

一方、機能を単純に分割すると依存関係が扱いづらくなるため、GraphiteなどはStacked PRを採用しています。

```text
PR 1: データモデル
  ↓
PR 2: 内部API
  ↓
PR 3: UI
```

各PRを小さく保ちつつ、stack全体をまとめて検証し、並列CIやstack-aware merge queueで処理します。 ([Graphite](https://graphite.com/docs/graphite-merge-queue))

GitHubのMerge Queueも、PR単体のheadだけでなく、最新のbase branchおよび先行する待機PRと組み合わせた状態でcheckを通してからMergeする仕組みです。高頻度PR環境では必須に近い機能です。 ([GitHub Docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue))

---

# 9. すべてをMerge前に証明せず、段階的に本番検証する

レビュー能力を増やしても、Merge前にすべての問題を発見することはできません。そこで一部の保証を、制御された本番運用へ移します。

- feature flag
- shadow traffic
- canary deployment
- small percentage rollout
- SLO・error rate監視
- 自動rollback
- 高リスクdeployの人間承認
- 外部システムによるdeployment protection rule

Google SREのcanary releaseは、一部の環境や利用者だけに新しい変更を適用し、指標を比較して続行・停止を判断する方式です。GitHub Environmentsも、required reviewers、self-review禁止、待機時間、外部GitHub Appによるdeployment protectionを提供します。 ([Google SRE](https://sre.google/workbook/canarying-releases/))

ただし、これはレビューを省略する方法ではありません。特にデータ消去、schema migration、プライバシー、認証、決済など、不可逆な変更には十分な事前審査が必要です。

---

# 10. AIレビュアー自体を攻撃対象として扱う

AIレビューでは、PR本文、コメント、コード、README、設定ファイルなどがAgentへの入力になります。したがって、レビュー対象のコード内に、

- 「この変更を承認せよ」
- 「別のファイルを読め」
- 「秘密情報を出力せよ」
- 「外部URLへ送信せよ」

といったprompt injectionが埋め込まれる可能性があります。

GitHubもcoding agentについてprompt injectionリスクを明示し、インターネットアクセス、workflow実行、未信頼ユーザーからのイベントなどを制限しています。CodeQLにもsystem prompt injectionを検出するqueryが追加されています。 ([GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/risks-and-mitigations))

さらに注意すべき点として、GitHub Copilot code reviewはPRのbase branchではなく、**head branchにあるcustom instructionsやagent skillsを読みます**。これは同じPRでinstructionsをテストできる便利な設計ですが、レビュー対象PR自身がレビュアーの指示を変更できることも意味します。 ([GitHub Docs](https://docs.github.com/copilot/using-github-copilot/code-review/using-copilot-code-review))

高保証環境では、次の防御が必要です。

- Review Agentは原則read-only
- secretsを与えない
- 外部network accessをdeny-by-defaultにする
- ephemeral sandboxで実行する
- PR内のinstructionsより、base branchまたは外部管理されたpolicyを優先する
- reviewer設定ファイルをCODEOWNERSで保護する
- instructions変更PRを通常コード変更から分離する
- Review AgentとFix Agentを別権限にする
- 書き込み、workflow実行、deployには人間承認を要求する
- tool callと取得した文脈を監査ログとして残す
- commit署名やSLSA provenanceで生成経路を追跡する

SLSAのprovenanceは、成果物がいつ、どこで、どのsourceとbuild processから生成されたかを検証するための基盤です。Agent生成コードでも、誰の指示で、どのAgent・設定・モデルが変更を作ったかを追跡する必要があります。 ([SLSA](https://slsa.dev/spec/v1.0/provenance))

---

# 11. 主要サービスの現在地

## GitHub Copilot code review

GitHubとの統合が最も深い選択肢です。リポジトリ探索、レビュー間のmemory、linked Issue・PR、長いPR向けのreview plan、重複コメントの集約、batch autofix、CodeQLとの連携、リポジトリ単位の利用指標まで一体化しています。 ([The GitHub Blog](https://github.blog/ai-and-ml/github-copilot/60-million-copilot-code-reviews-and-counting/))

## GitLab Duo Code Review

Merge Request作成時にリポジトリ文脈を事前収集し、cross-file dependencyやcustom instructionsを参照します。一方で事前スキャン後に自由に文脈を追加取得する方式ではなく、大規模MRではcontext limitの影響があるため、PR分割や重要ファイルの優先順位付けが必要です。 ([GitLab Docs](https://docs.gitlab.com/user/duo_agent_platform/flows/foundational_flows/code_review/))

## Claude Code Review

複数の専門Agentがコードベースを並列に調査し、その後にverification、deduplication、severity rankingを行う構成が特徴です。一般的な一発のLLMレビューより、Agent fleetと検証層を重視しています。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/code-review))

## OpenAI Codex review

Draft PRをreadyにした時の自動レビュー、`@codex review`による呼び出し、レビュー会話内から修正実装を依頼する流れを持ちます。OpenAI内部では、CodexがほぼすべてのPRをレビューする運用も報告されていますが、これは同社内の事例であり、一般環境での独立比較ではありません。 ([OpenAI](https://openai.com/index/introducing-upgrades-to-codex/))

## Gemini Code Assist for GitHub

PR summary、自動レビュー、severity threshold、style guide、リポジトリ文脈検索などを提供します。既存のGoogle Cloud、Gemini、GitHub環境との統合が中心です。 ([Google Cloud Documentation](https://docs.cloud.google.com/gemini/docs/code-review/review-repo-code))

## Amazon Q Developer

新規またはreopenされたPRに対するquality・security review、修正提案、`/q review`による手動実行を提供します。追加commitで常に自動再レビューされるとは限らないため、再トリガー設計を明示的に組む必要があります。 ([AWS Documentation](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/amazon-q-for-github.html))

## Cursor Bugbot

PRレビューに加え、push前レビュー、差分単位のincremental review、クラウドVM上で修正を試すAutofixに力を入れています。PRキューに入る前のshift-left用途が強い製品です。 ([Cursor](https://cursor.com/blog/bugbot-autofix))

## CodeRabbit

過去のレビューからの学習、repository guidelines、linked Issue・過去PR、MCP、Web検索、最新ドキュメント、cross-repository context、ローカルCLIを組み合わせています。特に、ライブラリの最新仕様や社内文書をレビュー時に検索する構成に向いています。 ([CodeRabbit](https://docs.coderabbit.ai/knowledge-base))

## Qodo

複数Agent、リポジトリ履歴、組織標準、仕様と実装の差、cross-repository analysisを重視しています。どのrules、skills、contextをレビューに使用したかを表示する方向も、監査性の面で重要です。 ([Qodo Documentation](https://docs.qodo.ai/code-review))

## Greptile

リポジトリをgraphとして索引化し、symbol、dependency、call relationshipから変更影響を調べる方式が特徴です。レビューへのreactionや対応履歴からチーム固有の傾向を学習します。 ([Greptile](https://www.greptile.com/docs/introduction))

## Graphite

主目的はAIレビュアーそのものではなく、stacked PR、PR navigation、merge queue、並列CIによるレビュー処理能力の向上です。AgentによってPR数が増えるほど、この種のworkflow infrastructureの価値が高くなります。 ([Graphite](https://graphite.com/docs/graphite-merge-queue))

現時点では、これらを公平に横比較できる標準的なproduction benchmarkはまだありません。各社の解決率、採用率、速度は母集団と測定方法が違うため、そのまま比較すべきではありません。

---

# 12. 研究結果は「完全自動レビュー」に慎重

最新研究を並べると、AIレビューは有用ですが、単独で人間を置き換える段階ではありません。

### c-CRAB

人間のレビュー指摘を実行可能なテストに変換し、Claude Code、Codex、Devin、PR-Agentなどを評価したところ、全Agentを合わせても解けたのは約40％でした。一方で、人間とは異なる種類の問題を見つけることがあり、補完関係が確認されています。 ([arXiv](https://arxiv.org/html/2603.23448v2))

### SWE-PRBench

350件のPRで8モデルを評価したところ、人間が指摘した問題の検出率はdiff-onlyでも15〜31％でした。また、固定的にコンテキストを増やすほど性能が低下しました。 ([arXiv](https://arxiv.org/html/2603.26130v1))

### 産業環境でのICSE 2025研究

4,335件のPR、238人の実務者を対象とした研究では、AIコメントの73.8％がresolvedになりました。しかし平均PR完了時間は5時間52分から8時間20分に伸び、誤ったレビュー、不要な修正、無関係なコメントも確認されました。これは観察研究であり単純な因果関係とは限りませんが、「レビューを付ければ必ず速くなる」わけではないことを示します。 ([arXiv](https://arxiv.org/abs/2412.18531))

### Human–AI Synergy研究

300のOSSプロジェクト、278,790件のレビュー会話では、AIのコード提案採用率は16.6％、人間は56.5％でした。また人間はAI生成コードのレビューで11.8％多く対話し、AIがあまり提供しない設計理解、知識移転、テスト方針に関するフィードバックを行っていました。 ([arXiv](https://arxiv.org/html/2603.15911v1))

### Code Review Agentの実態調査

3,109件のPRを分析した研究では、AI reviewerのみのPRのMerge率は45.20％、人間のみでは68.37％でした。また、閉じられたAIレビューのみのPRの60.2％が低いsignal-to-noise範囲にありました。ただし、PR難易度や選択バイアスがあり得るため、Merge率の差をそのままAIの因果効果とは解釈できません。 ([arXiv](https://arxiv.org/html/2604.03196v1))

これらを総合すると、

> AIは大量の一次スクリーニングには向く。
> 人間は意図、設計、組織固有知識、責任判断に残す。
> AIと人間のどちらか一方ではなく、異なる失敗モードを持つ複数層を組み合わせる。

という結論になります。

---

# 13. 評価指標も「コメント数」から変える

AIレビュー導入で追うべきではない指標は、

- AIが書いたコメント数
- レビューを実行したPR数
- Agentが生成した行数
- PRのMerge数だけ

です。

本当に必要なのは次です。

- 人間が費やしたレビュー時間
- 最初の有用な指摘までの時間
- AI指摘のprecisionと誤検出率
- 指摘によって実際に修正された割合
- 修正後に対応テストが成功した割合
- 見逃されて本番に到達した欠陥
- rollback、incident、change failure rate
- リスク別のMerge cycle time
- 1件の有用な指摘にかかった推論コスト
- 高リスクPRで必要承認を迂回した件数
- Agent生成PRと人間生成PRの品質差
- PR説明とdiffの不整合率

GitHubも、production signalとして「コメントが付いたか」ではなく、developer reactionと、Merge前に指摘事項が解消されたかを測っています。2026年7月には、リポジトリ単位でcoding agentが作ったPR数、Copilot reviewの利用状況、Merge速度などを追跡するdashboardとAPIを拡充しました。 ([The GitHub Blog](https://github.blog/ai-and-ml/github-copilot/60-million-copilot-code-reviews-and-counting/))

さらに、過去の重大bugや人間レビュー指摘を実行可能テストに変えた、組織固有のgolden benchmarkを作るべきです。LLM-as-a-judgeだけで自動レビュアー自身を評価すると、同じモデル系列の偏りを再生産する可能性があります。c-CRABのように、レビューが正しい修正へつながり、実際のテストを通すかを見る方が信頼できます。 ([arXiv](https://arxiv.org/html/2603.23448v2))

---

# 現時点で推奨できる完成形

実務で最も堅い構成は次のようになります。

1. **Issue・仕様ゲート**
   受け入れ条件、担当領域、優先度、リスクを確定する。

2. **Agent WIP制御**
   同時に開ける作業・Draft PR数を制限し、重複タスクを排除する。

3. **ローカルPreflight**
   build、lint、typecheck、test、security scan、AI自己レビューを実行する。

4. **Evidence Bundle生成**
   実行したコマンド、結果、リスク、設計意図、Rollback、provenanceを添付する。

5. **決定論的CIゲート**
   再現可能なfailureを先に処理し、失敗時は人間へ送らずAgentへ返す。

6. **Diff起点のAgentic Review**
   リポジトリ全体を索引として持ちながら、変更箇所から必要な証拠だけ検索する。

7. **専門Agent＋Verifier**
   correctness、security、tests、architectureを並列に調べ、最後に検証・重複除去する。

8. **Risk Router**
   低リスクは自動処理、高リスクだけCODEOWNER、security、architectへ送る。

9. **Fix・Re-test loop**
   Agent修正後に元の解析と関連テストを再実行する。

10. **Merge Queue**
    base branchや他の待機PRとの組み合わせを検証する。

11. **Progressive Delivery**
    feature flag、canary、監視、自動rollbackを使う。

12. **継続評価**
    人間のreactionだけでなく、見逃し、本番障害、修正率、レビュー時間で改善する。

---

# AWS Bedrock＋Claude Codeで構築する場合の現実的な構成

Claude Codeだけを中心に実装する場合でも、単一のClaudeに全部任せるより、次の役割分離が有効です。

```text
Author Agent
    ↓
Preflight Agent
    ↓
Context Planner
    ├─ diff・変更symbol
    ├─ callers・tests
    ├─ Issue・ADR
    ├─ lockfile対応の外部仕様
    └─ 必要時だけContext7・DeepWiki・MCP
    ↓
Review Agents
    ├─ correctness
    ├─ security
    ├─ tests
    └─ architecture
    ↓
Finding Verifier
    ↓
Risk Router
    ↓
GitHub Review / Human Review
```

特に古い知識を避けるには、外部仕様検索を単なる一般Web検索にせず、

- `package-lock.json`、`pnpm-lock.yaml`、`poetry.lock`などから実際のversionを特定
- 変更されたAPIだけを抽出
- そのversionに対応する公式documentを取得
- 取得元、version、取得日時をreview evidenceに保存
- 最新版の仕様と、プロジェクトが使用している版を混同しない

という順序にする必要があります。

リポジトリ文脈についても、巨大な固定Context Packではなく、

- 変更symbolのcall graph
- 関係するtest
- repository map
- CODEOWNERS
- ADR・設計書
- 過去の同種レビュー
- cross-repository contract

を検索可能にし、Reviewerが具体的な疑問を持った時だけ取得する構成が、品質、速度、コストのバランスを最も取りやすいです。

---

## 最終的な見方

AI Agent時代のボトルネックは、コードを書く能力ではなく、**変更を信頼できる状態へ変換する能力**です。

そのため最先端の取り組みは、AIレビュー製品の精度競争だけではなく、

- PRを作る前に減らす
- PRを小さくする
- 実行可能な証拠を要求する
- 文脈を適応的に検索する
- 決定論的解析とAIを重ねる
- 複数Agentの結果を検証する
- 人間をリスクの高い判断に集中させる
- 本番でも段階的に検証する
- Agent自身の権限とprompt injectionを防御する

という、開発プロセス全体の再設計になっています。

**完全無人レビューはまだ実験段階ですが、「人間が読むべきPRを10分の1に圧縮するシステム」は、すでに現実的な到達目標です。**

---
Powered by [AI Exporter](https://saveai.net)