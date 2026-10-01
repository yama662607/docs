そのまま別エージェントに渡せるように、調査済みの知見・一次ソース・未検証事項・実装目標・ベンチマーク方法まで含めた引き継ぎプロンプトにまとめます。前の調査結果を無条件に信じず、必ずソースコードで再検証するようにもしています。

高速 Code Review Bot の追加調査・設計・実装

あなたは、既存のAI Code Review Botのレビュー待ち時間を大幅に短縮するための調査・設計・実装担当エージェントです。

現在の最大の問題はレビュー品質ではなく latency です。

現状：

* 1つのPRレビューに 15分以上かかることがある
* レビュー完了待ちによって開発フローが止まる
* 品質を多少落としてでも、まずwall-clock latencyを大きく削減したい
* ただし、単純な品質劣化ではなく、OSSの優れた実装から速度改善の知見を取り込みたい

最重要KPIは、

PR作成・更新から最初の有用なレビュー結果が返るまでの時間

です。

理想としては通常PRで 1〜3分程度、少なくとも現在の15分超から大幅に短縮したいです。

⸻

1. 最初に行うこと

まず現在のreview botの実装を調査してください。

以下を特定してください。

1. PR取得
2. checkout / diff生成
3. changed file抽出
4. context構築
5. LLM呼び出し
6. tool call
7. review round
8. verification
9. summarization
10. GitHub comment投稿

のそれぞれについて、

* serial / parallel
* 呼び出し回数
* 使用model
* reasoning effort
* prompt token数
* output token数
* tool call回数
* retry
* timeout
* cache
* GitHub API call
* filesystem / grep / index処理

を確認してください。

特に、

T_total
├── T_fetch_pr
├── T_checkout
├── T_diff
├── T_filter
├── T_grouping
├── T_context
├── T_llm
├── T_tool
├── T_verify
├── T_summary
└── T_publish

のように分解できるようにしてください。

推測で最適化を始めるのではなく、

どこで何分使っているかをまず観測可能にすること

を優先してください。

⸻

2. 必ず調査してほしいOSS

以下のOSSについて、READMEだけでなく実際のソースコードを調べてください。

特にlatency削減に関係する実装を重点的に確認してください。

⸻

A. Alibaba OpenCodeReview

Repository:

https://github.com/alibaba/open-code-review

非常に重要です。

既に確認できている主要ソース：

architecture

https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/architecture.md

agent orchestration

https://github.com/alibaba/open-code-review/blob/main/internal/agent/agent.go

特に以下を読むこと：

dispatchSubtasks()
executeGroupSubtask()
applyResume()
reviewItemFingerprint()

grouping / selection

https://github.com/alibaba/open-code-review/blob/main/internal/agent/grouping.go

https://github.com/alibaba/open-code-review/blob/main/internal/agent/selection.go

template / thresholds

https://github.com/alibaba/open-code-review/blob/main/internal/config/template/template.go

https://github.com/alibaba/open-code-review/blob/main/internal/config/template/task_template.json

telemetry

https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/telemetry.md

CLI configuration

https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/cli-reference.md

⸻

Alibabaで既に見つかっている重要知見

1. Group単位parallel review

dispatchSubtasks() ではfile groupごとにgoroutineを生成し、semaphoreでconcurrencyを制御している。

デフォルト：

concurrency = 8

概念：

group A ─┐
group B ─┤
group C ─┼── parallel
group D ─┤
group E ─┘

現在のbotがserialなら最優先候補。

⸻

2. Semantic grouping

全PRを1つの巨大contextに入れず、関連fileをgroup化する。

一方、1 file = 1 callにも固定しない。

最大group file数は10。

調査して、

* grouping自体のLLM latency
* grouping overhead
* group sizeとreview latency
* semantic groupingが本当に必要か

を評価してください。

⸻

3. Small PRではLLM groupingを省略

GroupingPlan() を確認してください。

既定値：

GROUPING_MIN_FILES = 4
GROUPING_BUNDLE_LINE_THRESHOLD = 200

少数fileではLLMによるsemantic groupingを実行しない。

つまり、

small PR
→ deterministic grouping
large PR
→ LLM semantic grouping

というfast pathが存在する。

これを現在botへ応用できるか検討してください。

⸻

4. Plan phaseを条件付き実行

既定値：

PLAN_MODE_LINE_THRESHOLD = 50
PLAN_MODE_GROUP_LINE_THRESHOLD = 100

つまり、

* 大きいfile
* ある程度大きいgroup

だけplan phaseを実行する。

小さい変更は、

Plan

を丸ごと省略する。

現在botにplannerが常時存在する場合、重要な改善候補。

⸻

5. Review roundsを可変にする

OpenCodeReviewでは、

low    = 1 round
medium = 2 rounds
high   = 3 rounds

参考：

https://github.com/alibaba/open-code-review/blob/main/internal/config/template/effort.go

通常PRのfast pathでは1 roundにできるか検討。

⸻

6. Resume / incremental review

非常に重要。

applyResume() と reviewItemFingerprint() を調査してください。

diffからfingerprintを生成し、

以前review済み
+
diff fingerprint一致
↓
review resultを再利用

する。

つまりpushごとにPR全体を再レビューしない。

理想：

PR v1
A reviewed
B reviewed
C reviewed
PR v2
A unchanged → reuse
B changed   → review
C unchanged → reuse

これは実運用上非常に大きい可能性がある。

現在botにも、

(repo, path, diff hash, rule version, model/version)

などをkeyとするincremental cacheを導入できるか調べてください。

⸻

3. OpenAI Codex

Repository:

https://github.com/openai/codex

Codexの /review 実装はかなり公開されています。

重点的に読むこと。

Review session

https://github.com/openai/codex/blob/main/codex-rs/core/src/session/review.rs

Review task

https://github.com/openai/codex/blob/main/codex-rs/core/src/tasks/review.rs

Review request

https://github.com/openai/codex/blob/main/codex-rs/prompts/src/review_request.rs

Review rubric

https://github.com/openai/codex/blob/main/codex-rs/prompts/templates/review/rubric.md

⸻

Codexから注目している知見

Dedicated review execution path

通常agentに単純に、

review this

と言っているだけではない。

review専用thread / taskを作っている。

さらにreview時には不要な能力を制限している。

例：

* web search disabled
* cached web search disabled
* Goals disabled
* multi-agent disabled

調査ポイント：

現在botに不要なtools / MCP / browser / subagent capabilityが大量に渡されており、agentが無駄に探索していないか。

レビュー用tool surfaceを、

diff
read_file
grep
find_reference
submit_finding

程度まで削れる可能性を検討する。

⸻

Review model override

Codexには、

review_model

が存在する。

つまりcoding用modelとreview用modelを分離できる。

現在botでも、

strong coding model
≠
fast review model

にできないか検証。

重要なのはモデルのブランド名ではなく、

* TTFT
* output tokens/s
* reasoning latency
* tool-call latency
* review recall
* false positive

を実測すること。

⸻

4. Kodus

Repository:

https://github.com/kodustech/kodus-ai

重点ソース：

Finder

https://github.com/kodustech/kodus-ai/blob/main/libs/code-review/infrastructure/agents/core/finder.agent.ts

Verification orchestration

https://github.com/kodustech/kodus-ai/blob/main/libs/agent-harness/infrastructure/orchestration/verification-pass.ts

Context retrieval

https://github.com/kodustech/kodus-ai/blob/main/libs/code-review/infrastructure/agents/collaborators/rule-context.retriever.ts

⸻

Kodusから注目している知見

Candidate verificationをparallel化

runVerificationPass() を確認。

candidate findingごとにverifierを走らせるが、

Promise.all(...)

とbounded concurrencyを使っている。

default concurrency:

4

重要なのは、

verifierがPR全体を最初から再レビューするのではなく、candidate findingのみ確認する

構造。

現在botが、

review
↓
second full review
↓
critic full review

となっているなら非常に無駄。

理想：

fast review
↓
3 candidate findings
↓
3 findingsだけparallel verification

⸻

Contextを必要なときだけ取る

Kodusには contextNeed という発想がある。

例：

diff-only
full-file
symbol-references
sibling-file
cited-file

つまり最初からrepository全体を探索しない。

diffで判定可能
→ repository retrievalなし
diffだけでは判定不能
→ 必要なcontextだけ取る

このlazy context acquisitionは非常に重要。

現在botが全PRについて、

tree
grep
references
tests
related files

を無条件で取得していないか確認してください。

⸻

Prompt cache

Kodus finderには、

* static system promptをcache
* heavy re-runをparallel
* cached prefixを再利用

する設計が存在する。

特にAnthropic等prompt caching対応providerの場合、

static prefix
↓ cache
worker 1
worker 2
worker 3

の構造が使える。

現在botでもprompt prefix cachingを活用できるか検証。

⸻

5. Cloudflare security-audit-skill

Repository:

https://github.com/cloudflare/security-audit-skill

これは通常reviewより重いため、そのままhot pathへ入れないこと。

ただし以下の思想を参考にする。

hunter
↓
candidate
↓
independent verifier

つまり、

expensive verificationはfinding candidateだけに使う。

レビュー全体を二重に行わない。

fast / deep modeの分離も調査してください。

⸻

6. reviewdog

Repository:

https://github.com/reviewdog/reviewdog

LLM reviewerではないが重要。

考え方：

Ruff
ESLint
TypeScript
Semgrep
Clippy
custom analyzer
        ↓
reviewdog
        ↓
PR inline comments

調査目的は、

LLMで考えなくていいものを、deterministic checkへ移す

こと。

静的解析はLLM reviewと並列実行してください。

例：

parallel
├── Ruff
├── ESLint
├── typecheck
├── Semgrep
└── LLM review

LLMへは既にdeterministic checkが捕捉するstyle/lint issueを報告させない。

⸻

7. Semgrep / ast-grep

Semgrep:

https://github.com/semgrep/semgrep

ast-grep:

https://github.com/ast-grep/ast-grep

目的：

* rule-based prefilter
* cheap bug detection
* structural pattern matching
* LLM candidate絞り込み

への利用可能性を調査。

ただしdeterministic analysisを追加した結果、blocking latencyが増えないよう、

LLM reviewと並列に走らせること。

⸻

8. Anthropic Claude Code review implementation

Repository:

https://github.com/anthropics/claude-code

参考：

https://github.com/anthropics/claude-code/blob/main/plugins/code-review/commands/code-review.md

multi-agent review設計を確認する。

ただし今回の主目的は品質ではなく速度なので、

* agent数
* serial dependency
* pre-check
* filtering
* duplicated work

の観点から読む。

「良い設計だから採用」ではなく、

latencyを悪化させるmulti-agent構造は採用しない

こと。

⸻

9. 追加で探してほしいOSS

上記だけでなくGitHub上で、

AI code review
PR review agent
fast code review
parallel code review
incremental code review
cached code review
review agent
pull request reviewer

などを調査してください。

特に、

* source code公開
* starsが多い
* production利用実績がある
* recent commitがある
* latency optimizationに言及している

ものを優先。

新しい有望実装が見つかったら、

repository
star
language
architecture
latency technique
source file
relevant function/class

まで記録してください。

READMEの説明だけで終わらせず、原則ソースコードで確認してください。

⸻

10. 検討してほしい最終architecture

以下を仮説として検証してください。

                    PR webhook
                        │
                  diff generation
                        │
             deterministic filtering
                        │
              cheap deterministic router
                        │
            semantic grouping if needed
                        │
         ┌──────────────┼──────────────┐
         │              │              │
      group A         group B        group C
      fast LLM        fast LLM       fast LLM
         │              │              │
         └──────────────┼──────────────┘
                        │
                candidate findings
                        │
           bounded parallel verification
                        │
                  early publishing
                        │
               FAST REVIEW COMPLETE
                        │
                 optional deep pass
                  non-blocking

⸻

11. Fast path

通常PRはできれば、

1 review round
fast model
diff first
minimal tools
4〜8 parallel groups
no planner for small change
candidate-only verification

にする。

目標：

P50  < 2 min
P90  < 4 min

程度を一つの候補にする。

ただし現環境に合わせて現実的な値をbenchmarkから決める。

⸻

12. Deep path

以下だけをdeep review対象にすることを検討。

* auth
* payment
* crypto
* permissions
* database migration
* concurrency
* security-sensitive files
* very large PR
* generated candidate with high severity
* user explicitly requests deep review

Deep pathは可能ならblocking checkにしない。

⸻

13. Straggler問題

並列化だけでは、

A = 40 sec
B = 50 sec
C = 45 sec
D = 7 min

の場合、全体が7分になる可能性がある。

したがって、

worker completed
↓
result immediately publish

が可能か調査。

最終summaryを待たず、

partial findings

をGitHubへstream / batch publishする設計も検討。

少なくともfast review completionとdeep review completionを分離する。

⸻

14. ComputeとPublishを分離

必須候補。

悪い例：

LLM review 10 min
↓
GitHub comment API failure
↓
workflow retry
↓
LLM reviewをもう一度10 min

良い例：

review computation
↓
review-result.json
↓
persist
↓
publish

posting failure時は、

publishだけretry

する。

⸻

15. Cache

以下のcacheを検討。

Diff review cache

候補key：

repository
base SHA
head/path
normalized diff hash
review rule version
review prompt version
model family

ただしbase SHAが変わってもdiffが完全一致する場合のreuse可否も検討する。

⸻

Repository context cache

毎回同じ、

* AGENTS.md
* CLAUDE.md
* project conventions
* package metadata
* architecture summary
* symbol index

を読み直さない。

commit SHAやfile hashでinvalidaton。

⸻

Prompt cache

providerが対応している場合、

system prompt
review rubric
repository instructions
rules

をstable prefixとしてcache可能にする。

⸻

16. Cancellation

新しいpushが来た場合、

PR SHA A review

がまだ走っているのに、

PR SHA B

が来たなら、Aの残りreviewを中止できないか検討。

stale reviewへ10分使うのは無駄。

理想：

new head SHA detected
↓
cancel stale workers
↓
reuse unchanged completed shards
↓
review only changed shards

⸻

17. Early stop

agentが既に十分探索した後にもtool loopを続けていないか調べる。

候補：

submit_result
task_done
coverage threshold
no-new-findings
tool budget
time budget

などで早期終了できるようにする。

特に、

MAX_TOOL_REQUEST_TIMES = 100

のような「最大値」は安全弁であって、目標値ではない。

通常reviewでtoolを何十回も呼ぶのは避けたい。

⸻

18. モデルbenchmark

最低でも以下を比較。

同じPR corpusについて、

Model A
Model B
Model C

で、

* total review latency
* TTFT
* total output generation time
* token input
* token output
* tool call数
* findings recall
* false-positive rate
* severity precision

を測定。

高性能modelでも速度が数倍違う可能性がある。

重要：

model品質benchmarkではなく、Code Review workload上のlatency-quality frontierを見ること。

⸻

19. Concurrency benchmark

以下を最低限比較。

1
2
4
6
8
12

測定：

wall-clock
provider queue
429 rate
retry
cost
P50
P90
P99

「多ければ多いほど良い」と仮定しない。

provider rate limit / account limitsによって最適点を決める。

⸻

20. PR corpus

benchmark用に小・中・大を分ける。

例：

Small:
1〜3 files
<100 changed lines
Medium:
4〜15 files
100〜1000 changed lines
Large:
15+ files
1000+ changed lines

さらに、

buggy PR
clean PR
security PR
refactor
test-only
docs-only
generated file heavy

も含める。

⸻

21. 重要な評価指標

平均だけでなく、

P50
P90
P95
P99

を見ること。

特にCode Reviewでは、

10回に1回15分かかる

だけでも開発体験が悪い。

straggler analysisを必ず行う。

⸻

22. 実装優先順位の仮説

現時点の仮説は以下。

S

1. group/file reviewのbounded parallelization
2. incremental / resume review
3. fast review model
4. single review round fast path

A

5. lazy context retrieval
6. optional planning
7. candidate-only verification
8. small PR deterministic routing
9. prompt caching
10. stale run cancellation

B

11. static analysis parallelization
12. early partial publishing
13. compute/publish separation
14. deep review non-blocking化

この順番を鵜呑みにせず、現在botのprofiling結果に合わせて変更してください。

⸻

23. 最初に欲しい成果物

まず実装変更前に、

A. 現在architecture

PR
↓
...

の図。

B. latency breakdown

表：

Stage	Calls	Serial/Parallel	P50	P90	Tokens	Notes

C. bottleneck ranking

例えば：

1. sequential LLM reviewer
2. repeated repository search
3. verifier re-reading entire PR
4. no cache
5. slow posting

D. OSS比較

OSS	Technique	Relevant source	Expected impact	Difficulty

ソースURLを必ず付ける。

⸻

24. その後の実装

調査だけで終わらず、安全に変更できるものから実装する。

原則、

one optimization
↓
benchmark
↓
quality regression check
↓
next optimization

で進める。

巨大な一括rewriteは避ける。

⸻

25. 実装時の重要原則

Wall-clockを最優先

token costだけ下がっても、レビュー待ち時間が変わらなければ今回の目的には不十分。

品質を完全に捨てない

ただし、

15分で98点

より、

2分で90点
+
必要なときだけdeep review

を優先する。

全処理をblockingにしない

fast resultを先に返せる構造を優先。

Evidence based

「このほうが速そう」ではなくbenchmarkする。

⸻

26. 調査結果の引用方法

各主張には可能な限り、

Repository
File
Function / class
URL

を付ける。

例：

Alibaba OpenCodeReview:
internal/agent/agent.go
Agent.dispatchSubtasks()
https://github.com/alibaba/open-code-review/blob/main/internal/agent/agent.go

READMEだけの情報と、ソースコードで確認済みの情報を区別する。

⸻

27. 特に確認してほしい仮説

最終的に以下をYES/NOと根拠付きで答えてください。

1. 現botはLLM処理を十分parallelizeできているか
2. 1 PR = 1巨大agentになっていないか
3. file/group shardingした方が速いか
4. review roundを1回へ減らせるか
5. planningを小PRで省略できるか
6. strong modelを全reviewに使う必要があるか
7. review専用fast modelに置換できるか
8. verifierはPR全体でなくcandidateだけ見ればよいか
9. repository contextをlazy retrievalできるか
10. unchanged diffをcache/reuseできるか
11. new push時にstale reviewをcancelできるか
12. partial resultを先にpublishできるか
13. static analysisをreviewと並列化できるか
14. posting retryでLLM reviewを再実行していないか
15. prompt cachingが利用可能か

⸻

28. 最終目標

「非常に賢いreview agentを1つ作る」ことではありません。

最終目標は、

開発者を待たせないCode Review system

です。

理想は、

PR push
   ↓
数秒: pre-check開始
   ↓
30〜60秒: 最初の結果
   ↓
1〜3分: fast review completion
   ↓
必要ならdeep reviewが後から追加

というUXです。

OSSのarchitectureを参考にしながら、

quality / latency / costのPareto frontierを実測し、特にlatency側へ強く寄せてください。

⸻

参考ソース一覧

Alibaba OpenCodeReview
https://github.com/alibaba/open-code-review

Alibaba architecture
https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/architecture.md

Alibaba agent orchestration
https://github.com/alibaba/open-code-review/blob/main/internal/agent/agent.go

Alibaba selection
https://github.com/alibaba/open-code-review/blob/main/internal/agent/selection.go

Alibaba grouping
https://github.com/alibaba/open-code-review/blob/main/internal/agent/grouping.go

Alibaba template logic
https://github.com/alibaba/open-code-review/blob/main/internal/config/template/template.go

Alibaba default template
https://github.com/alibaba/open-code-review/blob/main/internal/config/template/task_template.json

Alibaba telemetry
https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/telemetry.md

Alibaba CLI reference
https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/cli-reference.md

OpenAI Codex
https://github.com/openai/codex

Codex review session
https://github.com/openai/codex/blob/main/codex-rs/core/src/session/review.rs

Codex review task
https://github.com/openai/codex/blob/main/codex-rs/core/src/tasks/review.rs

Codex review request
https://github.com/openai/codex/blob/main/codex-rs/prompts/src/review_request.rs

Codex review rubric
https://github.com/openai/codex/blob/main/codex-rs/prompts/templates/review/rubric.md

Kodus
https://github.com/kodustech/kodus-ai

Kodus finder
https://github.com/kodustech/kodus-ai/blob/main/libs/code-review/infrastructure/agents/core/finder.agent.ts

Kodus verification pass
https://github.com/kodustech/kodus-ai/blob/main/libs/agent-harness/infrastructure/orchestration/verification-pass.ts

Kodus context retriever
https://github.com/kodustech/kodus-ai/blob/main/libs/code-review/infrastructure/agents/collaborators/rule-context.retriever.ts

Cloudflare security audit
https://github.com/cloudflare/security-audit-skill

reviewdog
https://github.com/reviewdog/reviewdog

Semgrep
https://github.com/semgrep/semgrep

ast-grep
https://github.com/ast-grep/ast-grep

Anthropic Claude Code
https://github.com/anthropics/claude-code

Anthropic review plugin
https://github.com/anthropics/claude-code/blob/main/plugins/code-review/commands/code-review.md

PR-Agent
https://github.com/qodo-ai/pr-agent

akkie76 review skills
https://github.com/akkie76/code-review-skills

⸻

重要：

上記に書かれた既存調査結果も仮説として扱い、実装時点の最新main branchのコードを読んで再確認してください。古いREADMEや二次記事だけを根拠にしないでください。

この版は、別エージェントが「もう一度ゼロから背景を調べる」時間を減らしつつ、既存調査の誤りをそのまま実装へ持ち込まないよう、一次ソースの場所と検証事項をセットにしています。