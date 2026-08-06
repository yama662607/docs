> From: https://chatgpt.com/c/6a677ac0-c614-83e8-8b4d-611ee60d377c

# you asked

message time: 2026-07-28 00:35:28

Graphifyやserenaのmcp toolのように、リポジトリの探索を効率化するmcpやskillについて、最も今主流かつ将来性があり、あなたがいいと思うツールを調べてください。
広い範囲で調査しましょう

---

# chatgpt response

I’ll first map Graphify and Serena’s capabilities, then survey comparable MCPs, skills, and CLIs by adoption, maintenance, analysis method, agent compatibility, and long-term potential.

---

# chatgpt response

Early findings separate Serena’s precise symbol operations from Graphify’s repository-wide mapping. I’m now filtering GitNexus, Probe, ChunkHound, Octocode, and Codanna by practical analysis quality and operating cost.

---

# chatgpt response

The likely stack is Serena plus a persistent structure graph, with PR and multi-repository tools activated selectively. GitNexus remains promising, but its noncommercial OSS license limits adoption.

---

# chatgpt response

# 調査結果

2026年7月28日時点で、リポジトリ探索を効率化するMCP・CLI・Agent Skillを、解析方式、精度、更新追従性、導入実績、ライセンス、Claude Code／Codexとの互換性まで含めて比較しました。

## 結論

現時点での私の評価は、次のとおりです。

| 目的 | 最も薦めるもの |
|---|---|
| **1つだけ導入する** | **Serena** |
| **実務で最も強い構成** | **Serena + codebase-memory-mcp + GitHub MCP** |
| **リポジトリ全体の構造・影響範囲を理解する** | **codebase-memory-mcp** |
| **視覚的なコードグラフを使う** | **GitNexus**。ただし商用利用制限に注意 |
| **コード・ドキュメント・PDF・設計資料を一緒に地図化する** | **Graphify** |
| **今後伸びそうな新興ツール** | **CodeGraph** |
| **GitHub上の別リポジトリ・PR・npmまで横断調査する** | **Octocode** |
| **インデックスなしで素早く探索する** | **Probe** |
| **「なぜこの実装になったか」をGit履歴から調べる** | **ChunkHound** |

特に重要なのは、**Serena、Graphify、GitNexus、codebase-memory-mcpは競合というより、扱う問題が異なる**という点です。

- Serenaは「このシンボルの定義、参照、実装はどこか」
- codebase-memory-mcpやCodeGraphは「この変更がどこまで伝播するか」
- GitNexusは「全体構造をグラフとして見て、関連箇所を探索する」
- Graphifyは「コードだけでなく設計資料やPDFまで含めて、プロジェクトを地図化する」
- ChunkHoundは「いつ、なぜ、この設計になったか」
- Octocodeは「外部リポジトリやPRを含め、根拠付きで調査する」

という分担になります。

---

# なぜ単一ツールでは不十分なのか

コード探索には、実際には少なくとも次の5階層があります。

1. **文字列検索**  
   `rg`、grep、BM25など。「この文字列がどこにあるか」

2. **構文探索**  
   Tree-sitterなどのAST。「関数、クラス、import、呼び出しらしきものは何か」

3. **意味的なシンボル解決**  
   LSPやコンパイラ。「同名の関数のうち、実際に参照しているのはどれか」

4. **グラフ探索**  
   呼び出しグラフ、継承、依存関係、HTTPルート、変更のblast radius。

5. **時間・外部文脈の探索**  
   Git履歴、PR、Issue、外部ライブラリ、公式ドキュメント。

Tree-sitterだけでは動的ディスパッチ、DI、マクロ、reflectionなどを完全には解決できません。一方、LSPは正確ですが、「プロジェクト全体の中心ノード」「数段先までの影響範囲」「複数サービス間の接続」のような問い合わせには必ずしも向きません。

2026年の研究でも、軽量なcall／inheritance topologyをエージェントに与えると、関数レベルの位置特定が改善し、探索ラウンドが短くなった一方、構造情報を過剰に詰め込むと効果が逓減すると報告されています。つまり、**全グラフを常にプロンプトへ入れるのではなく、必要な部分だけ問い合わせる設計がよい**ということです。 ([arXiv](https://arxiv.org/abs/2606.26979))

また、codebase-memory-mcpの評価では、グラフ探索は通常のファイル探索よりトークンを約10分の1、ツール呼び出しを約2.1分の1にできた一方、最終的な回答品質は0.83対0.92で、完全なソース文脈や網羅的な文字列検索では通常探索が優位でした。論文自身も、**グラフ探索と通常のファイル読み取りを組み合わせるハイブリッド構成が最適**と結論づけています。なお、この評価は開発プロジェクト側によるもので、独立した横断ベンチマークではない点には注意が必要です。 ([arXiv](https://arxiv.org/html/2603.27277v1))

---

# 有力ツールの比較

## Sランク：常用する価値が高い

### 1. Serena

**総合1位。1つだけ入れるならこれです。**

SerenaはLSPを利用して、ファイルを丸ごと読むのではなく、シンボル単位でコードを探索・編集します。

主な操作は以下です。

- シンボル検索
- ファイル内のシンボル一覧
- 参照元シンボルの検索
- 宣言・実装への移動
- 診断情報の取得
- シンボル単位の挿入・置換
- rename
- 関数やクラスの前後への編集
- 長期ワークフロー用のmemory

Serenaの強みは、単なるAST推測ではなく、対応言語ではLSPが持つ意味情報を利用できることです。「`User`という名前が何十個もある」「interfaceと実装が分かれている」「import aliasがある」といった場合に、文字列検索より正確になります。 ([GitHub](https://github.com/oraios/serena))

Claude Code、Codex、Gemini CLI、OpenCode、Cursor／VS Code系など幅広いクライアントに対応し、MITライセンスです。一部の高度な型階層・依存パッケージ検索・リファクタリング機能はJetBrainsプラグイン側に寄っていますが、通常のlanguage-server版だけでも実用性は十分高いです。 ([GitHub](https://github.com/oraios/serena))

**弱点**

- 永続的な全体グラフではない
- LSPの起動やプロジェクト認識が不完全だと精度が落ちる
- 言語サーバーによって実装検索などの対応差がある
- 動的言語の実行時関係を完全には追えない
- アーキテクチャの可視化には弱い

それでも、「探索から編集まで」を1本で担えるため、**最初に導入すべき標準ツール**です。

---

### 2. codebase-memory-mcp

**永続的な構造グラフとして、現在最も薦めやすい選択です。**

Tree-sitter解析から関数、クラス、呼び出し、継承、HTTPルート、サービス間接続などを永続グラフに保存し、MCPから問い合わせます。現在のREADMEでは158言語のAST解析、主要言語に対するハイブリッドLSP解決、15個のMCPツール、単一静的バイナリ、MITライセンスを掲げています。 ([GitHub](https://github.com/DeusData/codebase-memory-mcp))

特に強い質問は次のようなものです。

- `ProcessOrder`を呼ぶ経路を入口まで追跡
- このクラスを変更すると影響するモジュール
- リポジトリ内の中心的な関数・クラス
- AからBまでの呼び出し経路
- 循環依存
- デッドコード候補
- サービス間のHTTP接続
- モノレポ内のパッケージ依存

自動インデックスとバックグラウンドwatcherがあり、既に登録したプロジェクトについてGitベースの変更検出を行えます。 ([GitHub](https://github.com/DeusData/codebase-memory-mcp))

**私がGitNexusより会社利用でこちらを優先する理由**

- MITライセンス
- 単一バイナリ
- ベクトルDBや外部LLMが必須ではない
- ヘッドレスでエージェントから使いやすい
- 構造問い合わせに集中したツール設計
- 自動更新に対応
- 複数クライアントへ持ち運びやすい

**弱点**

- 比較的新しいプロジェクト
- 研究評価は開発者自身によるもの
- C/C++マクロ、reflection、DI、実行時ディスパッチには限界がある
- グラフ結果を最終的な事実とせず、該当ソースを再確認する必要がある

Serenaが「IDEの目」なら、codebase-memory-mcpは「リポジトリ全体の索引と道路網」です。

---

## A+ランク：目的次第ではSランク

### 3. GitNexus

**個人利用・大学・非商用研究では、最も魅力的なグラフ探索ツールの1つです。**

GitNexusはTree-sitterでコードを解析し、LadybugDBへ永続化し、BM25、semantic search、RRFを組み合わせます。CLI／MCP版ではローカルの永続DBを使い、ブラウザ版ではWASMによってサーバーなしでグラフを生成します。視覚化はSigma.js／Graphologyを使用します。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus))

特徴は次のとおりです。

- インタラクティブな知識グラフ
- 関数・クラス・import・call chain
- コードクラスタ
- execution flow
- Graph RAGエージェント
- MCP + Skills + hooks
- Claude Code、Codex、Cursor、OpenCodeなどへのセットアップ
- read-only MCPモード
- リポジトリ単位の許可リスト

Claude CodeとCodexの両方に、MCP、Skill、hookをまとめて入れられる設計は非常に将来性があります。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus))

ただし、重大な注意点があります。GitNexusのOSS版はMITではなく、**PolyForm Noncommercial License 1.0.0**です。個人研究、学習、教育機関、公的研究組織などは許可されていますが、通常の企業・商用プロジェクトで無条件に使えるライセンスではありません。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus/blob/main/LICENSE))

また、README上では「変更ファイルのみを再インデックスする完全なincremental indexing」がまだロードマップ項目になっています。頻繁に変更される巨大モノレポで、常時最新のグラフとして使う場合は確認が必要です。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus))

**評価**

- 個人開発・大学研究：かなり薦める
- 企業利用：ライセンス確認が必要
- 人間もグラフを見る：非常に強い
- エージェントだけが使う：codebase-memory-mcpの方が単純

京都大学での研究用途や個人的なKanade／Shiori開発の探索には有力ですが、将来的にサービスとして商用化するコードベースでは慎重に扱うべきです。

---

### 4. Graphify

**GraphifyはSerenaの競合ではなく、アーキテクチャ理解用のAgent Skillです。**

GraphifyはコードをTree-sitterでローカル解析し、次のようなグラフを生成します。

- call／import／inherit関係
- God node
- Leiden法によるサブシステム分割
- ファイル間の意外な接続
- ADR、RFC、設計コメント
- SQLスキーマや設定ファイル
- PDF、画像、動画、音声
- コードとドキュメントの接続

コード部分はLLMなしでローカル処理され、ドキュメントやPDF、画像、動画の意味解析だけが、設定したモデルを利用します。関係には`EXTRACTED`、`INFERRED`、`AMBIGUOUS`の区別が付き、ソースから直接得た関係と推測された関係を判別できます。 ([GitHub](https://github.com/Graphify-Labs/graphify))

また、埋め込みやベクトルストアを使わず、実際のグラフを探索する方針です。 ([GitHub](https://github.com/Graphify-Labs/graphify))

**特に向いている用途**

- 初めて触る巨大リポジトリの全体把握
- アーキテクチャ図の生成
- オンボーディング資料
- コードとADR・設計書の対応付け
- 「このプロジェクトの中心概念は何か」
- 「境界づけられたコンテキストはどう分かれるか」
- コードだけでなく論文PDFや仕様書も含む研究プロジェクト

**向いていない用途**

- renameや正確な参照検索
- 毎回の小さなコード編集
- 常に最新のグラフをバックグラウンド維持すること
- LSPレベルの厳密な型・シンボル解決

OSS版はオンデマンドでグラフを作る構成で、全コンテキストを常時更新するalways-on層はEnterpriseとして案内されていますが、現在はwaitlist段階です。 ([GitHub](https://github.com/Graphify-Labs/graphify))

したがって、Graphifyを常時起動するMCPとして扱うより、**設計調査を始めるときに`/graphify` Skillとして呼ぶ**方が合理的です。

---

### 5. CodeGraph

**今後の最有力候補です。半年から1年後に順位が上がる可能性があります。**

CodeGraphはRustベースの解析カーネルを持ち、事前生成したグラフから、1回の問い合わせで以下を返すことを狙っています。

- 関連するソースコード
- entry point
- caller／callee
- call path
- blast radius
- 関連シンボル
- 変更によって影響を受ける範囲

「grep → ファイル読む → 次のシンボルを検索 → また読む」というループを、一度の構造問い合わせにまとめる設計です。 ([GitHub](https://github.com/colbymchenry/codegraph))

さらに、OSネイティブのファイル監視を使って変更を自動同期します。Webフレームワークについても、Django、FastAPI、Flask、Express、NestJSなどのURLルートとhandlerの接続を解析します。 ([GitHub](https://github.com/colbymchenry/codegraph))

**有望な点**

- MIT
- 100%ローカル
- Rustカーネル
- 自動同期
- 1回の問い合わせに集約
- framework-aware
- Claude Code、Codex、Cursor、Gemini CLI、OpenCodeなどに対応
- 変更影響分析を中心に設計

**慎重に見るべき点**

- かなり新しい
- ベンチマークはプロジェクト自身によるもの
- 言語によって解析の深さが異なる
- READMEの大きな性能数値を、そのまま他ツールとの公平な比較には使えない
- Graph、Serena、標準探索のどれを優先すべきか、実リポジトリで評価する必要がある

現時点では、Serenaとcodebase-memory-mcpを置き換えるより、**自分のリポジトリで並行試験する最有力候補**です。

---

## Aランク：専門用途で非常に強い

### 6. Octocode

Octocodeは「ローカルリポジトリの探索」だけでなく、GitHub上のリポジトリ、PR、npmパッケージまで同じツールセットで調査します。

内部では次を組み合わせています。

- ripgrep
- AST検索
- 正確な範囲読み取り
- LSP
- GitHub検索
- PR調査
- npm調査
- CLI
- MCP
- Agent Skill

つまり、Serenaのようなローカル意味探索と、GitHub MCPのような外部調査の中間に位置します。 ([GitHub](https://github.com/bgauryy/Octocode))

**最も向いているケース**

- 依存ライブラリの実装を調べる
- upstreamで同じ問題が修正されていないか調べる
- PRやIssueと現行コードを突き合わせる
- 複数リポジトリ間で実装パターンを比較する
- 「根拠となるファイルと行を示して調べる」

ユーザーが以前検討していた「PRレビュー時に最新ライブラリ仕様や外部リポジトリを参照させる」用途では、かなり相性がよいです。

---

### 7. Probe

Probeは、永続グラフを作らず、ripgrepの速度、Tree-sitterのAST、BM25／semantic retrievalを組み合わせて、関数やクラス単位のまとまったコンテキストを返します。 ([GitHub](https://github.com/probelabs/probe))

**強み**

- 事前インデックスがほぼ不要
- コードを単なるテキスト断片ではなく関数・クラス単位で返す
- MCPとCLIの両方
- 組み込みAgentを利用可能
- Claude CodeやCodexの認証を利用できる構成
- 大きなグラフDBを管理したくない場合に便利

**位置付け**

- Serenaより意味的な厳密さは弱い
- codebase-memory-mcpより多段の関係探索は弱い
- ただし導入が軽く、最初の候補絞り込みには強い

常時MCPとして公開するより、`repo-search` Skill内から必要時にCLIとして呼ぶ運用も適しています。

---

### 8. ChunkHound

ChunkHoundは「今のコード」だけでなく、Git履歴と外部Web調査を同じ探索系に入れます。

例えば、

- 認証処理が変わった理由
- v2.4以降にbillingがどう変化したか
- あるブランチ同士で、なぜ同じ箇所が異なる実装になったか
- 最近30コミットでwebhook処理に何が起きたか
- 外部APIの仕様変更が現在のバグに関係するか

といった質問に向きます。 ([GitHub](https://github.com/chunkhound/chunkhound))

regex検索はAPIなしで使えますが、semantic searchやdeep researchではembedding／LLMの設定が必要です。 ([GitHub](https://github.com/chunkhound/chunkhound))

Serenaが「何がどこにあるか」、ChunkHoundが「なぜそうなったか」を担当する関係です。

---

# 補助ツール

## GitHub公式MCP

GitHub MCPは、ローカルコードのAST解析用ではありません。

主に以下を担当します。

- リポジトリとファイル
- コード検索
- コミット
- PR
- Issue
- GitHub Actions
- Release
- Dependabot／security alerts
- Discussionや通知

したがってSerenaやcodebase-memory-mcpと競合せず、**履歴とコラボレーション文脈を補完するもの**です。 ([GitHub](https://github.com/github/github-mcp-server?utm_source=chatgpt.com))

機能が多いため、全toolsetを公開するのではなく、通常は例えば次のように絞るべきです。

```text
repos
pull_requests
issues
actions
code_security
```

公式READMEも、必要なtoolsetだけを有効にすると、LLMのツール選択とコンテキストサイズを改善できると説明しています。 ([GitHub](https://github.com/github/github-mcp-server?utm_source=chatgpt.com))

探索エージェントにはread-onlyにし、write／merge／delete系を原則公開しない構成が安全です。

---

## Repomix

Repomixは探索エンジンというより、選択したリポジトリ内容をAI向けの単一ファイルへまとめるツールです。

- `.gitignore`対応
- token count
- Tree-sitter圧縮
- Git diff
- commit log
- 特定ファイルだけのpack
- watch mode
- MCP／Skill

に対応しています。 ([GitHub](https://github.com/yamadashy/repomix?utm_source=chatgpt.com))

次の用途では今でも有用です。

- ChatGPTなどMCPを使えない環境へリポジトリを渡す
- レビュー用context packを作る
- 特定の関連ファイルだけを一度に渡す
- エージェント間の静的な受け渡し成果物を作る

ただし、継続的なシンボル探索ではSerenaやグラフ系に劣ります。

---

## Understand Anything

Understand Anythingは、複数エージェントでリポジトリを解析し、ファイル、関数、クラス、依存関係の知識グラフと視覚的dashboardを生成する方向のツールです。 ([GitHub](https://github.com/Egonex-AI/Understand-Anything?utm_source=chatgpt.com))

Graphifyより「人間向けのインタラクティブdashboard」に寄っています。一方で、初期解析にLLMとトークンを多く使う構成になりやすいため、毎日の低コスト探索より、オンボーディングやプレゼン用として評価するのが適切です。

---

## Codanna／CodeGraphContext

CodannaはRustベースのローカルコードインテリジェンスで、1回のMCP呼び出しからsymbol context、call graph、impact analysisを返す設計です。codebase-memory-mcpやCodeGraphの軽量競合として注視する価値があります。 ([GitHub](https://github.com/bartolli/codanna?utm_source=chatgpt.com))

CodeGraphContextはCLIとMCPを備え、callers、callees、class hierarchy、call chainをグラフ化し、live file watchingにも対応しています。既に一定の導入実績がありますが、現時点では上位候補と機能が重なります。 ([GitHub](https://github.com/codegraphcontext/codegraphcontext?utm_source=chatgpt.com))

この2つを、Serenaやcodebase-memory-mcpと同時に常用する必要はありません。比較実験の候補です。

---

# 私が実際に採用する構成

## 構成A：最も薦める標準構成

```text
常時有効
├── Serena
├── codebase-memory-mcp
└── GitHub MCP（read-only、toolset限定）

必要時のみSkill／CLIから使用
├── Graphify
├── Octocode
├── ChunkHound
└── Probeまたはrg
```

### 役割分担

| 問い合わせ | 使用するもの |
|---|---|
| 名前や概念から候補を探す | `rg`、Probe、Octocode |
| 正確な定義・参照・実装 | Serena |
| caller、callee、依存、影響範囲 | codebase-memory-mcp |
| PR、Issue、Actions、コミット | GitHub MCP |
| 変更理由や過去の設計意図 | ChunkHound |
| upstream・外部ライブラリ調査 | Octocode |
| 全体設計・ドキュメントを地図化 | Graphify |
| 最終的な事実確認 | 実際のソースとテスト |

これが、速度、精度、ライセンス、将来性のバランスが最もよい構成です。

---

## 構成B：個人開発・大学研究

```text
Serena
+ GitNexus
+ Graphify
+ GitHub MCP
```

GitNexusの視覚グラフを人間も利用し、Serenaで正確なシンボル操作、Graphifyでコード・論文・仕様書をまとめて把握します。

特に研究コードでは、

- Pythonシミュレーション
- 解析notebook
- 実験プロトコル
- 論文PDF
- パラメータ設定
- 実験画像処理
- 結果ドキュメント

が分断されがちなので、Graphifyのマルチモーダルな地図化は通常のコードグラフより価値があります。

---

## 構成C：PRレビュー専用

```text
Serena
+ codebase-memory-mcp または CodeGraph
+ GitHub MCP
+ Octocode
+ ChunkHound
```

レビュー時の流れは次のようにします。

1. GitHub MCPでdiff、PR説明、関連Issueを取得
2. Serenaで変更シンボルの定義と参照を確認
3. グラフでcaller、callee、route、test、blast radiusを取得
4. ChunkHoundで対象周辺の変更理由を確認
5. Octocodeで依存ライブラリの現在の実装・upstream PRを確認
6. 実ソースとテストで検証
7. 根拠ファイル・シンボル・コミットを添えてレビュー

これにより、「diffだけを見て誤検知する」「古いライブラリ知識で指摘する」「変更されていない遠方の影響を見落とす」という問題をかなり減らせます。

---

# MCPを増やしすぎない方がよい

探索用MCPを5個も6個も常時公開すると、エージェントは次のような問題を起こします。

- 同じ目的の似たツールから誤ったものを選ぶ
- tool descriptionだけでコンテキストを消費する
- 同じ検索を複数ツールで繰り返す
- どのインデックスが最新か分からなくなる
- グラフ間で異なる結果が返る
- サーバー起動や監視プロセスが増える

そのため、**常時有効なMCPは最大3個程度**にして、用途が重なるツールはAgent SkillからCLIとして必要時に呼ぶ構成がよいです。

Agent Skillsは、Claude CodeとCodexの双方がオープンなAgent Skills形式を採用しています。Claude CodeではSkill本文を必要時だけ読み込むprogressive disclosureが使われ、Codexでも`SKILL.md`とscripts／referencesをまとめた再利用可能なワークフローとして扱われます。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/skills))

したがって、将来性の高い設計は、**特定MCPに全ロジックを埋め込むことではなく、MCPとCLIを選択する薄いSkillを上に置くこと**です。

---

# 推奨する`repo-explore` Skillの設計

Skillには、次のルーティング規則を入れるのがよいです。

```text
1. まず質問を分類する

A. 正確なシンボル・参照・編集
   → Serena

B. 呼び出し経路・依存関係・変更影響
   → codebase-memory-mcp または CodeGraph

C. 文字列・エラーメッセージ・曖昧な概念
   → rg / Probe / Octocode

D. PR・Issue・コミット・CI
   → GitHub MCP

E. 過去の設計理由
   → ChunkHound + git blame/log

F. 全体設計・サブシステム・資料との接続
   → Graphify

G. 外部リポジトリ・依存ライブラリ
   → Octocode + GitHub MCP

2. グラフ結果だけで結論を出さない
3. 最終候補の実ソースを読む
4. 動的挙動はテスト・実行・ログで確認する
5. 回答にはファイル、シンボル、根拠を付ける
```

さらに、巨大リポジトリでは、

```text
最大探索深度
最大返却シンボル数
対象ディレクトリ
除外ディレクトリ
生成コードの扱い
vendorの扱い
testを含めるか
```

をSkill側で制御します。研究が示すように、巨大なグラフを無制限に返すより、必要な逆参照や近傍だけ返す方が効果的です。 ([arXiv](https://arxiv.org/abs/2606.26979))

---

# 最終順位

## 総合

1. **Serena**  
   最も成熟しており、日常の探索・編集に直結する。

2. **codebase-memory-mcp**  
   永続グラフ、MIT、単一バイナリ、自動更新というバランスがよい。

3. **GitNexus**  
   非商用なら非常に強力。可視化とGraph RAGが魅力。ただしライセンスが最大の制約。

4. **Graphify**  
   アーキテクチャ理解とコード以外の資料統合では独自性が高い。常用MCPではなくオンデマンドSkill向き。

5. **CodeGraph**  
   技術設計は非常に有望。今後の本命候補だが、まだ実プロジェクトでの検証が必要。

6. **Octocode**  
   GitHub、PR、npm、ローカルコードを横断する調査で強い。

7. **Probe**  
   インデックスなしで始める軽量探索として優秀。

8. **ChunkHound**  
   Git履歴と外部仕様を含む「理由の調査」で強い。

## 私の最終推薦

現在すぐに組むなら、次の構成が最も堅実です。

```text
Serena
+ codebase-memory-mcp
+ GitHub MCP（read-only・toolset限定）
+ repo-explore Agent Skill
```

そしてSkillから必要時だけ、

```text
Graphify
Octocode
ChunkHound
Probe
```

を呼びます。

個人・大学研究でグラフを自分でも眺めたい場合は、codebase-memory-mcpに加えて、または置き換えてGitNexusを試す価値があります。企業利用や将来の商用化を想定するリポジトリでは、GitNexusのライセンスを避け、**Serena + codebase-memory-mcpまたはCodeGraph**を中心にするのが安全です。

この分野の将来は、単一の巨大な「コード理解MCP」ではなく、**LSPによる正確なシンボル操作、軽量な永続グラフ、通常のソース探索、Git・外部情報を、Agent Skillが問い合わせごとに選び分ける構成**に向かうと考えます。

---

# you asked

message time: 2026-07-28 01:28:34

調査ありがとうございます。
あなたの提案する各構成について、全く使用しない場合よりも使った方がメリットが大きいと言えるかどうか、積極派と否定派の双方の意見をたくさん集めて評価してください。

---

# chatgpt response

I’ll compare each proposed stack against standard grep, file reading, and Git workflows, weighing benchmarks, research, issues, user praise and criticism, operational costs, and misdirection risks.

---

# chatgpt response

Early evidence favors targeted hybrid use: LSP layers often add value, while always-on graph stacks can lose benefits through stale indexes, dynamic-code errors, duplication, and overhead.

---

# chatgpt response

Independent research mainly supports structural graphs as a method, not vendors’ dramatic token-savings claims; product benchmarks therefore need corroboration from real-world defect reports.

---

# chatgpt response

# 結論

前回の提案を再評価すると、**「追加のMCPやSkillは、入れない場合より常に有利」とは言えません**。

最も証拠に合う結論は、次のようになります。

| 構成 | 「何も追加しない」場合との比較 | 最終評価 |
|---|---:|---|
| 薄い`repo-explore` Skillだけ | 多くの場合プラス | **採用推奨** |
| Serenaだけ | 大規模・型付き・横断参照が多いならプラス | **条件付きで推奨** |
| グラフMCPを1つだけ | 影響範囲・呼び出し経路の調査ではプラス | **対象タスク限定で推奨** |
| **A：Serena + codebase-memory + GitHub MCPを常時有効** | 大規模リポジトリではプラスの可能性、小規模では過剰 | **構成を軽量化すべき** |
| **B：Serena + GitNexus + Graphify + GitHub MCPを常時有効** | 日常開発では重複と陳腐化が上回りやすい | **常用は非推奨** |
| Bをオンボーディング・設計調査時だけ使う | 大規模・複雑なプロジェクトでは明確に有益 | **用途限定で推奨** |
| **C：PRごとにSerena + graph + GitHub + Octocode + ChunkHound** | 普通のPRでは過剰、高リスクPRでは有益 | **段階的エスカレーションなら推奨** |
| 複数のコードグラフMCPを同時常用 | 重複、誤選択、インデックス不整合が増える | **非推奨** |

したがって、前回の「Serena + codebase-memory-mcp + GitHub MCPを常時有効」という案は、少し積極的すぎました。

私の修正版は、

```text
標準ツール
+ 薄い探索ルーティングSkill
+ 意味解析レイヤーを1つ
+ 必要時だけグラフを1つ
+ 必要時だけGitHub・履歴・外部リポジトリ調査
```

です。

---

# 1. 比較対象となる「何も使わない」の定義

比較を正確にするため、基準を2段階に分けます。

## B0：純粋な標準構成

- Claude Code／Codexなどの標準ファイル読み取り
- `rg`、grep、find
- shell
- Git
- テスト、ビルド、lint
- エージェント自身の推論

追加MCP、インデックス、コードグラフ、独自Skillは使いません。

## B1：現代的な標準構成

B0に加えて、クライアント公式またはIDE側のコードインテリジェンス／LSP機能を使います。

この違いは大きいです。Claude Codeの公式資料も、コードインテリジェンスを使えば、1回の定義ジャンプがgrepと複数ファイル読み取りを置き換えられると説明しています。また、`gh`のようなCLIはGitHub MCPよりコンテキスト効率がよいとされています。現在はMCPスキーマも遅延ロードされるため、「MCPを登録しただけで全定義がコンテキストを埋める」という以前の批判は弱まりましたが、不要なMCPを無効にすることは今も推奨されています。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/costs))

つまり、**SerenaをB0と比べると有利になりやすい一方、既に十分なネイティブLSPがあるB1と比べると、追加価値は小さくなります。**

---

# 2. 積極派の主張はどこまで正しいか

## 2.1 「コードの構造を与えると探索精度が上がる」

これは概ね正しいです。

ICLR 2025のRepoGraphは、4種類のソフトウェアエンジニアリング手法にコードグラフを追加し、平均相対改善率32.8%を報告しました。ただし、表を見ると絶対値での解決率向上は各構成でおよそ2.0〜2.66ポイントで、トークン数と推論費用は多くの構成で増えています。したがって「32.8%改善」という数字だけから、一般的なMCPでも大幅改善すると解釈するのは危険です。 ([ICLR Proceedings](https://proceedings.iclr.cc/paper_files/paper/2025/file/4a4a3c197deac042461c677219efd36c-Paper-Conference.pdf))

ACL 2025のLocAgentも、専用に学習したQwen-2.5-Coder-32Bとグラフ誘導を組み合わせ、ファイル位置特定で最大92.7%、複数回試行の問題解決でPass@10を12%改善しました。これはグラフによる多段探索が有効になり得る強い証拠ですが、専用エージェント、専用学習、専用グラフを組み合わせた結果であり、GitHubからMCPを1つ追加するだけの条件とは異なります。 ([ACL Anthology](https://aclanthology.org/2025.acl-long.426/))

さらにISSTA 2026採択論文では、強いgrep-first Codexに軽量なcall／inheritance情報を追加すると、

- Func@5が2.2ポイント向上
- 探索が平均1.6ラウンド短縮
- 中規模リポジトリでPass@1が3.4ポイント向上
- 実行間の分散がおよそ半減
- 入力トークンは約10%増加

という結果でした。大幅な知能向上というより、**探索経路を安定させる効果**が中心です。 ([arXiv](https://arxiv.org/html/2606.26979v2))

### 評価

積極派の「構造情報は価値がある」は正しいです。

ただし、最も再現性の高い効果は、

- 正解率が劇的に上がること
- 人間のようにコードを完全理解すること

ではなく、

- 参照先の見落としを減らす
- 探索経路を短くする
- 実行ごとのばらつきを減らす
- トークンを節約する

ことです。

---

## 2.2 「グラフはトークンとツール呼び出しを大幅に減らす」

これも、構造的な質問に限定すればかなり正しいです。

codebase-memory-mcpの開発者による評価では、グラフエージェントと通常のファイル探索エージェントを比較して、

| 指標 | グラフ | 通常探索 |
|---|---:|---:|
| 回答品質 | 0.83 | 0.92 |
| ツール呼び出し | 2.3回 | 4.8回 |
| トークン | 約1,000 | 約10,000 |

となりました。

グラフはhub検出、callerランキング、構造横断に強い一方、ソース全文の理解、網羅的grep、マクロを多用するCでは通常探索に負けています。論文自身も、**構造質問にはグラフ、ソースレベルの確認にはファイル探索というハイブリッドが最適**と結論づけています。 ([arXiv](https://arxiv.org/html/2603.27277v1))

ただし、この評価はプロジェクト側によるもので、回答評価にも開発者が関与しています。独立したSWE-bench評価ほど強い証拠ではありません。

CodeGraphも、7リポジトリを使った自社評価で、コスト60%減、トークン69%減、ツール呼び出し89%減を主張しています。ただし、各リポジトリでアーキテクチャ質問を1問ずつ比較したプロジェクト内ベンチマークであり、日常の修正、デバッグ、テスト、レビュー全体に一般化できる数字ではありません。 ([GitHub](https://github.com/colbymchenry/codegraph/blob/main/README.md))

### 評価

積極派が言うほど大きな節約が出る場合はありますが、対象は主に、

- 「この関数を呼ぶ入口は何か」
- 「変更の影響範囲はどこか」
- 「この2つのシンボルはどう接続されるか」
- 「中心的な依存ノードは何か」

といった**関係問い合わせ**です。

「この実装は正しいか」「この境界条件でバグらないか」「このマクロ展開後に何が起きるか」では、グラフだけに頼ると品質が下がります。

---

## 2.3 「LSPをエージェント向けに公開するSerenaは有益」

Serenaの機能自体には明確な独自価値があります。

- 定義検索
- 参照元検索
- 実装検索
- 診断
- rename
- シンボル本体の置換
- シンボル前後への挿入
- 40以上の言語サーバー対応

などを、エージェント向けの高水準なMCP操作として公開します。 ([GitHub](https://github.com/oraios/serena))

利用者からも、「grep、ast-grep、計装スクリプトを自作していたがSerenaが目的に合致した」「複雑なプロジェクトではgame changerだった」という報告や、プロダクションアプリでSkillsとの組み合わせが大きな違いを生んだという報告があります。 ([Reddit](https://www.reddit.com/r/ClaudeAI/comments/1lfsdll/try_out_serena_mcp_thank_me_later/))

ただし、Serena公式READMEに掲載された「エージェント自身による評価」はプロジェクト提供の評価プロンプトによるものであり、独立評価ではありません。機能の存在を確認する資料としては有効ですが、効果量の証拠としては弱めに見るべきです。 ([GitHub](https://github.com/oraios/serena))

---

# 3. 否定派の主張はどこまで正しいか

## 3.1 「今のエージェントはgrepだけでも十分強い」

これは否定派の最も強い論拠です。

ISSTA 2026の研究では、同じモデルと制約で比較した予備実験において、グラフ誘導のLocAgentはFunc@5が59.5%、grep-first Codexは83.2%でした。また、任意で呼べるcall-graph MCPを20件のSWE-bench問題に与えたところ、エージェントはほとんど使わず、通常のgrepを選びました。 ([arXiv](https://arxiv.org/html/2606.26979v2))

これは「グラフが無価値」という意味ではありません。むしろ、

> エージェントにグラフツールを与えただけでは、適切に使われるとは限らない

という意味です。

同じ研究では、グラフ専用エージェントへ置き換えるより、通常のgrep結果に軽量な構造ヒントを添える方が有効でした。つまり、**grepを捨ててグラフへ移行するのではなく、grep-firstを補強する方が堅い**ということです。

---

## 3.2 「グラフを使う段階を間違えると、むしろ悪化する」

これは非常に重要です。

2026年のSeeRepo研究では、グラフ表現を問題の位置特定段階で使うと、GPT-5-miniでPass@1が55.0%から55.4%へわずかに向上し、入力トークン25%、費用26%を削減しました。

しかし同じグラフを、

- 修正段階で使うとPass@1が50.0%まで低下
- パッチ検証段階で使うと51.6%まで低下

しました。研究者は、修正時に広い依存情報を見せると、無関係な周辺コードへ注意が逸れ、変更範囲を不必要に広げやすいと分析しています。 ([arXiv](https://arxiv.org/html/2606.14061v3))

### 評価

グラフの適切な用途は、

```text
問題理解
→ 候補ファイルの特定
→ 入口・caller・影響範囲の把握
```

までです。

その後は、

```text
実ソースを読む
→ 最小修正
→ テスト
→ 実行時確認
```

へ戻すべきです。

「修正前後に必ずグラフを見ろ」という強制hookは、逆効果になる可能性があります。

---

## 3.3 「ツールを増やすほどエージェントは賢くなるわけではない」

これも否定派が正しい部分です。

一般的なツール選択研究では、Claude Sonnet 4.6に常に5ツールを見せる場合の正しいツール選択率は87.1%でしたが、平均2.2個に絞った適応的な提示では93.1%になりました。中難度問題では60.9%対76.8%まで差が広がっています。 ([arXiv](https://arxiv.org/html/2605.24660v2))

別のToolScope研究も、意味的に重複したツールの統合と絞り込みによって、複数ベンチマークでツール選択精度が8.38〜38.6%改善したと報告しています。これはコード探索専用の研究ではありませんが、**似たMCPを大量に並べると選択が曖昧になる**という原理を支持します。 ([arXiv](https://arxiv.org/html/2510.20036v2))

現在のClaude CodeではMCPスキーマが遅延ロードされるため、登録した全ツールの詳細が毎回プロンプトを占有するわけではありません。しかし、ツール名の重複、どれを使うかの判断、サーバープロセス、インデックス更新といったコストは残ります。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/costs))

---

## 3.4 「インデックスは古くなり、静かに不完全な結果を返す」

これも現実的な問題です。

### Serena

利用者の中には、約6万行で小さく整理されたファイルと詳細なdocstringを持つコードベースでは、Serenaを追加してもClaude Codeが改善せず、ファイルを直接読ませる方が簡単だったという報告があります。 ([Reddit](https://www.reddit.com/r/ClaudeCode/comments/1qlz0em/my_claude_code_setup_after_8_months_12_custom/))

また過去の報告には、

- 小規模ディレクトリでも遅い、メモリを多く使う
- C#／C++／Rust混在環境で約30GBを消費して停止
- LLVMの7,749ファイルのインデックスに約11時間

といったものがあります。これらは全利用者に共通する頻度を示すものではありませんが、LSPやプロジェクト構造によって最悪ケースが生じることを示しています。 ([Reddit](https://www.reddit.com/r/ClaudeAI/comments/1lfsdll/try_out_serena_mcp_thank_me_later/))

### codebase-memory-mcp

利用者からは「コード検索が実際に改善した」という評価と同時に、複数リポジトリを並行運用した際の高CPU使用率が報告されています。さらに、2026年6月の大規模リポジトリ向けumbrella issueには、インデックスが終了しない、タイムアウトする、メモリが50GB超まで増えるといった問題がまとめられています。 ([GitHub](https://github.com/DeusData/codebase-memory-mcp/issues/45))

### GitNexus

GitNexusでは、5,000ファイル超のリポジトリで「解析成功」と表示されながらFTSが作成されず、検索結果が空になる問題や、read-only MCPモードで`query`、`context`、`impact`が無言で空になる問題が報告されました。これらは既にクローズされていますが、**成功表示だけではインデックス完全性を保証できない**という典型的な失敗形態です。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus/issues/1715))

### CodeGraph

CodeGraphでは、Windows Junctionを使ったmulti-root構成で、本来8,798ファイルあったインデックスが3ファイルまで縮小しながら、`Index is up to date`と表示されたという報告があります。また、過去には12,016ファイルのプロジェクトで約52,000個のファイルディスクリプタを保持する問題も報告され、後者は修正済みです。 ([GitHub](https://github.com/colbymchenry/codegraph/issues/1033))

### 評価

GitHub Issuesは「この問題が何%の利用者に起きるか」を示しません。しかし、次のようなfailure modeが現実にあることは示します。

- インデックスが部分的
- 古い
- 空
- 成功扱いなのに不完全
- watcherが追従しない
- リソースを過剰消費する

このため、グラフ結果を最終的な事実として扱うのは危険です。

---

# 4. 構成Aの再評価

## 元の構成

```text
Serena
+ codebase-memory-mcp
+ GitHub MCP
```

## 積極派の見方

この3つは理論上、役割が分かれています。

| ツール | 独自の役割 |
|---|---|
| Serena | 正確なシンボル、参照、実装、rename |
| codebase-memory | caller、依存、経路、blast radius |
| GitHub MCP | PR、Issue、Actions、コミット、セキュリティ情報 |

大規模な型付きリポジトリで、

- 複数モジュールを跨ぐ
- 同名シンボルが多い
- PRやIssueを頻繁に参照する
- 変更影響の調査が多い

なら、B0より有利になる可能性は高いです。

## 否定派の見方

問題は、実際にはかなり重複することです。

- SerenaとネイティブLSPが重複
- Serenaとcodebase-memoryの参照・caller探索が一部重複
- GitHub MCPと`gh` CLIが重複
- 3サーバーの起動、更新、障害切り分けが必要
- グラフの不完全性をエージェントが見抜けない

特にClaude Codeでは、公式に`gh` CLIの方がMCPよりコンテキスト効率がよいと説明されています。GitHub MCPを使う場合も、GitHubは`repos`、`issues`、`pull_requests`などのtoolsetや個別ツールを限定できる設計にしています。 ([Claude Platform Docs](https://docs.anthropic.com/en/docs/claude-code/costs))

## 判定

### B0との比較

中規模以上なら、**使った方が有利になる可能性が高い**です。

### B1との比較

ネイティブコードインテリジェンスと`gh`が既に機能しているなら、3つすべてを常時有効にする優位は明確ではありません。

### 修正版A

```text
常時有効
├── 標準のRead / rg / Git / test
├── repo-explore Skill
└── 意味解析を1つ
    ├── ネイティブcode intelligence
    └── またはSerena

必要時
├── codebase-memory-mcp または CodeGraphのどちらか1つ
└── GitHub MCPまたはgh
```

**構成Aは、軽量化すれば「使わないより有利」と評価できます。元の3つ常時構成は条件付きです。**

---

# 5. 構成Bの再評価

## 元の構成

```text
Serena
+ GitNexus
+ Graphify
+ GitHub MCP
```

## 積極派の見方

この構成は、オンボーディングやアーキテクチャ理解には非常に強力です。

GitNexusは、

- call chain
- blast radius
- execution flow
- cluster
- API route map
- グラフ可視化

などを持ち、人間とエージェントの両方が利用できます。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus))

Graphifyはコードだけでなく、

- ADR、設計文書
- SQL
- 設定ファイル
- PDF
- 画像
- 動画、音声

を同じグラフに接続でき、関係を`EXTRACTED`、`INFERRED`、`AMBIGUOUS`として区別します。研究コード、実験資料、論文、解析コードが混在するプロジェクトでは独自性があります。 ([GitHub](https://github.com/Graphify-Labs/graphify))

## 否定派の見方

日常開発ではGitNexusとGraphifyの役割が重なります。

両方とも、

- 構造グラフ
- 依存関係
- cluster／community
- アーキテクチャ説明
- エージェント向け問い合わせ

を提供するため、どちらの結果を信頼するか、どちらが最新かという問題が生じます。

Graphifyの利用者からも、

- グラフがほとんど呼ばれずgrepで十分だった
- 16〜20倍のコンテキスト削減を感じた
- 初期理解には正確だが、実装を始めるとすぐ古くなる
- live codingより、作業開始時のquery mapとしてよい

という賛否両方の報告があります。特に「変更した瞬間にグラフが古くなるため、実装中はLSPやgrepへ戻るべき」という意見は合理的です。 ([Reddit](https://www.reddit.com/r/ClaudeCode/comments/1tlbog1/are_tools_like_graphify_actually_placeboshype/))

Graphifyでは過去に、約400ファイルのmonorepo更新でsubagentの書き込み失敗により、約16分と約29万トークンを失った報告もあります。修正済みの個別問題ですが、グラフ生成そのものが失敗すると節約効果が逆転し得ます。 ([GitHub](https://github.com/Graphify-Labs/graphify/issues/808))

加えてGitNexusのOSS版はPolyForm Noncommercialであり、個人研究、教育機関、公的研究組織などには使用できますが、通常の商用利用を無条件に許可するMITライセンスではありません。 ([GitHub](https://github.com/abhigyanpatwari/GitNexus/blob/main/LICENSE))

## 判定

### 4つを常時有効

**使わない場合より不利になる可能性が高い**です。

理由は、

- グラフが二重
- 更新も二重
- ツール選択が曖昧
- live coding中の陳腐化
- GitNexusのライセンス制約
- SerenaとネイティブLSPの重複

です。

### 作業開始時だけ使う

これは明確に有益になり得ます。

修正版は次です。

```text
通常の実装
└── ネイティブcode intelligence または Serena

アーキテクチャ理解
├── コード中心・視覚化重視 → GitNexus
└── コード + PDF + ADR + SQL + 資料 → Graphify

GitHub作業
└── gh または限定GitHub MCP
```

**GitNexusとGraphifyは、同時常用ではなく目的に応じて片方を選ぶべきです。**

京都大学での非商用研究やKanade／Shioriの全体構造を一度把握する用途では有益ですが、毎回の実装セッションで両方を動かす必要はありません。

---

# 6. 構成Cの再評価

## 元の構成

```text
Serena
+ codebase-memory または CodeGraph
+ GitHub MCP
+ Octocode
+ ChunkHound
```

## 積極派の見方

高リスクPRでは、それぞれ異なる穴を埋めます。

| ツール | PRレビューでの役割 |
|---|---|
| Serena | 変更シンボルの参照、実装、型、diagnostics |
| graph | 影響範囲、caller、API経路、関連テスト |
| GitHub | PR本文、Issue、レビュー、Actions |
| Octocode | upstream、外部リポジトリ、npm、過去PR |
| ChunkHound | Git履歴、変更理由、release間の差分 |

Octocodeは、ローカル検索、AST、LSPに加え、GitHubコード、PR履歴、npmパッケージを1つの調査系にまとめています。 ([GitHub](https://github.com/bgauryy/Octocode))

ChunkHoundは、現在のコードだけでなく、特定コミット、タグ、branch range、直近Nコミットと外部技術情報を合わせて調査できます。「この変更がなぜ入ったか」「v2.4以降に何が変わったか」のような問いでは独自価値があります。 ([GitHub](https://github.com/chunkhound/chunkhound))

FSE 2026のSWR-Benchでは、完全なプロジェクト文脈を含む手動検証済み1,000 PRで、現在の自動レビューシステムは十分な性能に達していない一方、複数レビューの集約によってF1が最大43.67%改善しました。したがって、単一視点より複数の独立した調査視点を組み合わせる方向性には根拠があります。 ([Researchr](https://conf.researchr.org/details/fse-2026/fse-2026-research-papers/78/SWR-Bench-Assessing-LLM-Performance-in-Real-World-Code-Review-Comment-Generation))

## 否定派の見方

ただし、「複数レビューが有効」と「5つのMCPを毎回同時に見せる」は別です。

普通のPRで全ツールを動かすと、

- 同じコードを複数経路で再検索
- review latency増加
- トークン消費増加
- 複数グラフ由来の誤検知
- 古いupstream情報との混同
- 重要でない歴史的文脈への脱線
- コメント数だけが増える

可能性があります。

SeeRepoの結果も、グラフは位置特定段階では有益でも、修正・検証段階で広い依存情報を追加すると性能が落ちることを示しています。 ([arXiv](https://arxiv.org/html/2606.14061v3))

## 判定

### 全PRでフル構成

**使わない場合より不利になりやすい**です。

### 高リスクPRで段階的に使う

**使った方が有利になる可能性が高い**です。

推奨する段階構成は次です。

| レベル | 条件 | 使用するもの |
|---|---|---|
| 0 | 小さな通常PR | diff、実ソース、テスト、Git履歴 |
| 1 | 複数ファイル・型変更 | ネイティブLSPまたはSerena |
| 2 | 複数モジュール・API変更 | グラフを1つ |
| 3 | 外部ライブラリ・upstream依存 | Octocode |
| 4 | 過去の設計理由・regression疑い | ChunkHound |
| 5 | 高リスク・大規模変更 | 独立レビューを複数回行い集約 |

グラフはレビューの最初に、

```text
変更シンボル
→ caller
→ downstream
→ API boundary
→ 関連テスト
```

を絞るために使い、その後の正誤判定は実ソース、テスト、実行結果に戻します。

---

# 7. Serenaを1つだけ導入する案の再評価

前回は「1つだけならSerena」としましたが、これは少し広すぎる結論でした。

## Serenaが明確に有利になりやすい条件

- クライアントに十分なagent-facing LSPがない
- 型付き言語
- 同名シンボルが多い
- interfaceと実装が分離
- cross-file renameが多い
- 何十ファイルにも参照が広がる
- 複数のAIクライアントで同じ操作を使いたい

この場合はB0より有利になりやすいです。

## Serenaの追加価値が小さい条件

- 小規模リポジトリ
- ファイルが短く整理されている
- docstringと命名が詳細
- ネイティブcode intelligenceが既に安定
- 主な作業がMarkdown、設定、単純なテキスト修正
- LSP起動が重い言語や巨大プロジェクト

この場合、B1にSerenaを重ねる価値は明確ではありません。

## 修正した結論

「1つだけなら常にSerena」ではなく、

> **意味解析が足りない環境ならSerena、既に足りているなら追加しない**

が正確です。

PR／Issueの文脈取得が主目的なら、Serenaより`gh`または限定GitHub MCPの方が独自価値が大きい場合もあります。

---

# 8. codebase-memoryとCodeGraphは、どちらかを常用すべきか

現時点では、**どちらか1つを無条件に全リポジトリへ導入する証拠は不足しています**。

## codebase-memory

積極材料：

- 構造問い合わせでトークンとツール呼び出しを大幅削減
- 通常探索との役割分担が明確
- グラフ質問に向いている

慎重材料：

- 自社評価では通常探索より回答品質が低い
- 大規模リポジトリのCPU、メモリ、終了しないインデックス問題
- 静的構造なのでreflectionや実行時挙動を扱えない

## CodeGraph

積極材料：

- MCPツールを集約しており、ツール選択が比較的単純
- native watcher
- 入口、関連シンボル、コード片、影響範囲を一度に返す
- プロジェクト内評価では大きなコスト削減

慎重材料：

- 評価がプロジェクト自身によるもの
- multi-root、submodule、junction構成の不完全性
- 過去に重大なファイルディスクリプタ問題
- 開発速度が速く挙動が安定途上

### 結論

```text
codebase-memory OR CodeGraph
```

であって、

```text
codebase-memory AND CodeGraph
```

ではありません。

実リポジトリで比較して、正確に動いた方だけを採用すべきです。

---

# 9. 最も費用対効果がよいのは「薄いSkill」

今回の調査から、最も低リスクで有益なのは、巨大な検索システムではなく、**エージェントに探索方法を選ばせる薄いSkill**です。

内容は次程度で十分です。

```text
1. エラー文字列・曖昧な概念
   → rg / grep

2. 正確な定義・参照・実装
   → native code intelligence または Serena

3. caller・経路・影響範囲
   → グラフを1つ

4. PR・Issue・CI
   → gh または GitHub MCP

5. upstream・依存ライブラリ
   → Octocode

6. 過去の設計理由
   → git log / blame / ChunkHound

7. 全体構造・資料との関係
   → Graphify または GitNexus

8. 最終判断
   → 実ソース・テスト・実行結果
```

重要なのは、次の禁止事項を含めることです。

```text
- グラフを必ず使わせない
- grepを禁止しない
- 複数グラフで同じ調査を繰り返さない
- グラフだけを根拠に修正しない
- インデックスが完全だと仮定しない
- 修正段階で無関係な依存情報を広げない
```

ツール数を減らし、問い合わせに応じて適切なものだけ見せる考え方は、ツール選択研究の結果とも一致します。 ([arXiv](https://arxiv.org/html/2605.24660v2))

---

# 10. 実リポジトリでの採用判定方法

一般論だけでは最終判断できないため、リポジトリごとに小さな比較試験を行うのが最も確実です。

## 比較するタスク

同じモデル、同じ指示、別セッションで、最低20件を比較します。

| 種類 | 件数例 |
|---|---:|
| 定義・参照・実装の特定 | 5 |
| caller・影響範囲・経路 | 5 |
| 実際のバグ調査 | 5 |
| PRレビュー | 5 |
| 履歴・upstream調査 | 5 |

## 測定するもの

- 正解したか
- 必要なファイルを漏らさなかったか
- 不要なファイルを大量に読まなかったか
- 誤ったレビュー指摘を増やさなかったか
- ツール呼び出し数
- 入力トークン
- 経過時間
- CPUとメモリ
- インデックスが古い、空、部分的だった回数
- ツールを与えたのに実際に使われた割合

## 私が採用条件にする基準

これは研究上の普遍的基準ではなく、実務上の目安です。

```text
1. 正解率を下げない
2. 対象タスクで時間を20%以上、
   または入力トークンを30%以上減らす
3. 既知のシンボルを確実に検索できる
4. ファイル編集後に更新される
5. branch切替後に古い結果を返さない
6. silent empty / partial successを起こさない
7. CPU・メモリ・保守負担が節約効果を上回らない
```

グラフツール導入時には、少なくとも次をhealth checkに含めるべきです。

```text
rg --filesで得られる対象ファイル数との比較
既知の関数のcaller確認
既知の継承関係の確認
1ファイル編集後の更新確認
branch切替確認
submodule / symlink / generated code確認
空結果と「本当に存在しない」の区別
```

---

# あなたの用途に対する最終提案

Kanade／Shioriのような、複雑で長寿命、複数モジュール、PR頻度が高いプロジェクトでは、追加ツールの価値は比較的高いです。ただし、前回のフル構成ではなく次の形が適しています。

```text
常時
├── 標準のrg / Read / Git / tests
├── repo-explore Skill
└── native code intelligence または Serena

必要時
├── codebase-memory または CodeGraph
├── gh または限定GitHub MCP
├── Octocode：依存・upstream調査時
├── ChunkHound：履歴調査時
└── Graphify：設計書・PDFを含む全体地図作成時
```

研究リポジトリでは、Graphifyを安定したcommitやtagに対して定期的に実行し、コード、論文PDF、実験プロトコル、設定、解析資料の関係を整理する価値があります。しかし実装中の最新状態は、Serena／LSPと通常検索で追う方が安全です。

PRレビューでは、全ツールを毎回動かすのではなく、PRの規模と危険度に応じて追加調査を解放します。

## 最終順位

1. **薄い探索ルーティングSkill**  
   最も低コストで、使わない場合より有利になりやすい。

2. **ネイティブcode intelligenceまたはSerenaのどちらか1つ**  
   大規模・型付き・横断参照の多いリポジトリで有利。

3. **グラフMCPを1つ、位置特定と影響分析に限定して使う**  
   条件付きで有利。修正・検証まで支配させない。

4. **GitHub MCPは`gh`で不足する場合だけ、read-only・toolset限定で使う**  
   PR中心の運用では有利だが、常にMCPである必要はない。

5. **Graphify、GitNexus、Octocode、ChunkHoundはオンデマンド**  
   特定用途では非常に有益だが、常時並べると逆効果。

6. **複数の意味解析・複数のコードグラフ・複数のGitHub検索を常時重ねる構成**  
   何も追加しない場合より不利になる可能性が高い。

最も確からしい設計原則は、**「強いgrep-first標準構成を捨てず、意味解析を1つ、構造グラフを必要な段階だけ、特殊調査をSkillからオンデマンドで呼ぶ」**です。

---
Powered by [AI Exporter](https://saveai.net)