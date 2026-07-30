> From: https://chatgpt.com/c/6a6749f6-84d8-83ee-b0a5-c3d18c6bbccb

# you asked

message time: 2026-07-27 21:08:10

Atrasianのconfluenceとjiraというサービスについて調査して。特に、高度な自動化やapi連携、マクロ、mcpなどについて知りたいです。

---

# chatgpt response

Atlassianの公式情報を中心に、Confluence/Jiraの役割からAutomation、REST/Forge、マクロ、MCPまで整理し、実運用の構成例もまとめます。

「Atrasian」ではなく、正式には **Atlassian（アトラシアン）** です。

結論から言うと、**Jiraは「構造化された仕事・状態・依存関係の管理基盤」**、**Confluenceは「文章・設計・議事録・知識の管理基盤」**です。
この2つを組み合わせると、

- Jiraで「何を、誰が、いつまでに、どの状態で進めるか」を管理
- Confluenceで「なぜやるか、どう設計するか、何が分かったか」を管理
- Automation、REST API、Forge、MCPで両者と外部システムを接続

という構成を作れます。現在のAtlassian Cloudは、単なるタスク管理・Wikiではなく、**かなり拡張性の高い業務データ基盤**になっています。([Atlassian](https://www.atlassian.com/software/confluence?utm_source=chatgpt.com))

# 1. JiraとConfluenceの違い

| 項目 | Jira | Confluence |
|---|---|---|
| 主な役割 | タスク・課題・バグ・要求の管理 | 文書・知識・設計・議事録の管理 |
| データの中心 | Work item、ステータス、担当者、期限、フィールド | ページ、Live Doc、ブログ、ホワイトボード、データベース |
| 得意な検索 | JQL | CQL、全文検索、Rovo Search |
| ワークフロー | 非常に強い | 比較的軽量 |
| 自動化 | 課題作成、更新、遷移、通知、分岐 | ページ作成、整理、通知、アーカイブ |
| 拡張UI | Issue Panel、Custom Field、Dashboardなど | Macro、Page Action、Full Pageなど |
| AI連携 | 課題の検索・作成・更新 | 文書検索・要約・生成・更新 |

典型的には、Confluenceの設計書からJiraのEpicやTaskへリンクし、Jiraの進捗をConfluence上のレポートやマクロで可視化します。

---

# 2. 標準Automation

JiraとConfluenceには、ノーコードまたはローコードの **Atlassian Automation** が組み込まれています。

基本構造は次の通りです。

```text
Trigger
  ↓
Condition
  ↓
Branch / If-else
  ↓
Action
```

Jiraでは、課題作成、フィールド変更、コメント追加、ステータス変更、スケジュール、Webhook受信などをトリガーにできます。条件、分岐、課題作成、フィールド更新、サブタスク作成、通知などを組み合わせられます。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/jira-automation-triggers/?utm_source=chatgpt.com))

Confluenceでも同じ考え方で、ページ作成、更新、ラベル変更、スケジュール、Webhookなどから処理を開始できます。スペース単位のルールと、サイト全体を対象にするグローバルルールがあります。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/what-is-confluence-automation/?utm_source=chatgpt.com))

## Jira Automationの具体例

### PRレビュー管理

```text
GitHubからWebhookを受信
  ↓
repository、branch、authorを判定
  ↓
JiraにReview Taskを作成
  ↓
重大度や変更ファイルから担当チームを決定
  ↓
Confluenceの関連設計書をコメントに追加
  ↓
Slackへ通知
```

### ステータス同期

```text
Pull Request merged
  ↓
対応Jira課題を「レビュー済み」に遷移
  ↓
リリースノート候補ラベルを追加
  ↓
親Epicの進捗率を再計算
```

### SLA・期限管理

```text
毎朝実行
  ↓
期限が3日以内の未完了課題をJQLで抽出
  ↓
担当者ごとに分岐
  ↓
Slackまたはメールで通知
  ↓
期限超過なら優先度を上げる
```

## Smart Values

Automationでは、テンプレート変数に相当する **Smart Values** を使えます。

```text
{{issue.key}}
{{issue.summary}}
{{issue.assignee.displayName}}
{{issue.parent.key}}
{{triggerIssue.comments.last.body}}
{{now.plusDays(7)}}
```

Smart Valuesには文字列処理、日付計算、数値計算、条件式、リスト処理などがあり、Incoming WebhookやWeb RequestのJSONも参照できます。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/jira-smart-values-issues/?utm_source=chatgpt.com))

高度な分岐では、WebhookのJSON配列やコメント一覧をループできます。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/jira-automation-branches/?utm_source=chatgpt.com))

## 外部APIとの接続

Automationには主に次の2方向があります。

### 外部 → Atlassian

**Incoming Webhook Trigger** にJSONを送ります。

```json
{
  "repository": "kanade",
  "pullRequest": 314,
  "risk": "high",
  "relatedIssues": ["KAN-101", "KAN-205"]
}
```

JiraまたはConfluence側で受け取り、Smart Valuesを通して処理できます。Incoming WebhookはJiraとConfluenceのルールビルダーから設定できます。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/configure-the-incoming-webhook-trigger-in-atlassian-automation/?utm_source=chatgpt.com))

### Atlassian → 外部

AutomationのWeb Request系アクションから、GitHub、Slack、自作API、AWS Lambda、GitHub Actionsなどを呼び出します。

ただし、複雑な認証、長時間処理、大量データ、厳密なリトライ、状態管理が必要になると、Automationだけでは不十分です。その場合はForgeか外部バックエンドを使います。

## Automationの制約

Automationには月間実行数とサービス上の制限があります。プランによって上限が異なり、毎月リセットされます。また、自己再帰的なルールにはループ検出があり、遅い外部APIを大量に呼ぶルールには処理時間制限もあります。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/automation-service-limits/?utm_source=chatgpt.com))

Confluence Premiumは、公式説明上、ユーザー1人あたり月1,000回相当を組織全体でプールする方式です。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/view-your-confluence-automation-usage/?utm_source=chatgpt.com))

したがって、Automationは次の用途に適しています。

- 単純なイベント駆動処理
- 通知
- ステータス・フィールド更新
- 軽量な外部API呼び出し
- 日次・週次の整理
- 人間承認を含む業務フロー

一方、次にはForgeや独自サービスが向きます。

- AIによる長時間処理
- 大量の課題・ページの走査
- 複雑なトランザクション
- 厳密な再試行・キュー制御
- 独自UI
- 独自データモデル
- 複数組織・複数サイト対応

---

# 3. Jira・ConfluenceのREST API

## Jira REST API

Jira Cloudの中心はREST API v3です。

代表的な操作は次の通りです。

```text
GET    /rest/api/3/issue/{issueKey}
POST   /rest/api/3/issue
PUT    /rest/api/3/issue/{issueKey}
POST   /rest/api/3/issue/{issueKey}/comment
GET    /rest/api/3/search/jql
POST   /rest/api/3/issue/{issueKey}/transitions
```

課題、コメント、添付ファイル、プロジェクト、ユーザー、ワークフロー、フィールドなどを操作できます。([developer.atlassian.com](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/?utm_source=chatgpt.com))

検索にはJQLを使います。

```text
project = KAN
AND statusCategory != Done
AND labels = ai-review
AND updated >= -7d
ORDER BY priority DESC
```

## Confluence REST API

Confluenceでは主に次を操作します。

- ページ・ブログの検索
- ページ作成・更新
- コメント
- 添付ファイル
- ラベル
- スペース
- Content Property
- マクロ本体
- ページ階層

Confluenceのマクロ本体をMacro IDから取得し、保存形式や別の表現に変換するAPIもあります。([developer.atlassian.com](https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content---macro-body/?utm_source=chatgpt.com))

Confluenceでは本文表現が少し複雑です。単純なMarkdownではなく、用途に応じて次の形式が登場します。

- Atlassian Document Format、ADF
- Storage Format
- Atlas Doc Format相当のJSON
- HTML変換表現

APIでページを書き換える場合、**本文全体を取得してバージョン番号を上げて更新する**処理が必要になることがあります。AIに直接更新させる際は、同時編集の競合対策が重要です。

## 認証方式

### API Token

個人用スクリプトや内部試作に簡単です。

```bash
curl \
  -u "user@example.com:${ATLASSIAN_API_TOKEN}" \
  -H "Accept: application/json" \
  "https://example.atlassian.net/rest/api/3/issue/KAN-101"
```

ただし、利用者ごとの権限委譲や一般公開アプリには向きません。

### OAuth 2.0 3LO

複数ユーザーが使用する外部アプリでは、OAuth 2.0 Authorization Code Grant、いわゆる3LOが推奨されています。APIは `api.atlassian.com/ex/jira/<cloudId>/...` を通して呼びます。([developer.atlassian.com](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/?utm_source=chatgpt.com))

### Forge認証

Forgeアプリでは、マニフェストにスコープを宣言し、ユーザーまたはアプリとしてJira・Confluence APIを呼びます。

---

# 4. Confluenceのマクロ

Confluenceの **Macro** は、ページ内に埋め込む動的コンポーネントです。

標準マクロには、概念的には次のようなものがあります。

- 目次
- Jira課題一覧
- ページプロパティ
- ページプロパティレポート
- 展開・折りたたみ
- コードブロック
- ステータス
- タスク一覧
- チャート
- Smart Link
- 外部コンテンツ埋め込み

ただし、Cloud Editorへの移行に伴い、旧エディター時代の一部レガシーマクロは廃止・置換が進んでいます。たとえば旧Info、Tip、Note、Warningマクロは、Cloud EditorではPanel要素が相当機能を担います。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/insert-the-info-tip-note-and-warning-macros/?utm_source=chatgpt.com))

## 独自マクロ

Forgeを使うと独自マクロを作れます。

例：

```text
/architecture-status
/research-experiment
/pr-review-summary
/agent-run
/evidence-bundle
```

ページ上では、たとえば次のように表示できます。

```text
PR #314
Risk: High
Review agents: 4/5 completed
Blocking findings: 2
Related Jira: KAN-101
Last synchronized: 17:42
```

マクロには設定画面を持たせられます。

```yaml
modules:
  macro:
    - key: pr-review-summary
      resource: main
      resolver:
        function: resolver
      title: PR Review Summary
      config:
        resource: config
```

ユーザーはマクロ設定で、次のような値を指定できます。

- Jira Project
- GitHub Repository
- PR番号
- 表示対象
- 自動更新間隔
- 集計方法
- 表示形式

ForgeのUI KitまたはCustom UIを利用でき、設定可能なConfluenceマクロを公式手順で構築できます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/add-configuration-to-a-macro-with-ui-kit/?utm_source=chatgpt.com))

## マクロにデータを保存する方法

主な候補は以下です。

### マクロ設定

そのマクロ固有の軽い設定。

### Content Properties

ページやコンテンツにJSON形式のメタデータを付けます。

```json
{
  "project": "kanade",
  "documentType": "architecture-decision",
  "canonical": true,
  "reviewStatus": "approved"
}
```

Content PropertyはCQL検索用にインデックス化できます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/manifest-reference/modules/confluence-content-property/?utm_source=chatgpt.com))

### Forge Storage

アプリ固有データ。

### 外部DB

Supabase、PostgreSQL、DynamoDB、R2など。大規模データ、履歴、ベクトル検索にはこちらが向きます。

## マクロの重要な制約

Confluenceのマクロは、巨大なバックエンド処理そのものを実行する場所というより、**ページ内の表示・操作面**として使う方が適切です。

推奨構造は次の通りです。

```text
Confluence Macro
      ↓
Forge Resolver
      ↓
Forge Storage / Atlassian REST API
      ↓
Forge Remote
      ↓
AWS / Vercel / 独自バックエンド
```

---

# 5. Forge

**Forge** は、Atlassian Cloud用の公式アプリ開発基盤です。

Forgeでは次のような拡張ができます。

## Jira側

- Issue Panel
- Issue Action
- Issue Glance
- Custom Field
- Dashboard Gadget
- Full Page
- Project Page
- Workflow Validator
- Workflow Condition
- Workflow Post Function
- Automation Trigger
- Automation Action

## Confluence側

- Macro
- Full Page
- Content Action
- Space Page
- Space Settings
- Context Menu
- Content Byline Item
- Custom Content

Forgeのモジュールでは、Jiraカスタムフィールド、課題画面のUI、Confluenceのマクロやカスタムコンテンツ、Webhook、スケジュール実行などを定義できます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/modules/?utm_source=chatgpt.com))

同じForgeアプリから、Confluence MacroとJira Issue Panelを提供し、両方のAPIを呼ぶこともできます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/build-an-app-compatible-with-confluence-and-jira/?utm_source=chatgpt.com))

## UI KitとCustom UI

### UI Kit

Atlassian提供コンポーネントでUIを作ります。

利点：

- 実装が簡単
- Atlassian UIと馴染む
- セキュリティ面を管理しやすい
- 保守コストが低い

### Custom UI

Reactなどで自由な画面を作れます。

利点：

- 独自ダッシュボード
- 複雑なグラフ
- React Flow
- タイムライン
- エージェント実行状況
- インタラクティブなアーキテクチャ図

JiraとConfluenceのFull Page Moduleでは、UI KitまたはCustom UIを使ってAPI連携されたダッシュボードを作れます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/build-a-dashboard-app-with-the-jira-full-page-module?utm_source=chatgpt.com))

## Forgeのバックエンド機能

- Atlassian API呼び出し
- Key-Value Store
- Custom Entities
- Queue / Consumer
- Scheduled Trigger
- Web Trigger
- 外部API呼び出し
- Secrets
- 外部OAuth
- Forge Remote

Scheduled Triggerは、現時点の公式仕様では、時間・日・週単位の定期実行を提供します。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/manifest-reference/modules/index-common/?utm_source=chatgpt.com))

## Forge Remote

Forge Remoteを使うと、Forgeと自分のAWS、Vercel、Cloud Runなどを接続できます。

```text
Jira / Confluence UI
       ↓
Forge
       ↓ signed request
AWS API Gateway
       ↓
Lambda / ECS / Bedrock / Claude
       ↓
Result
       ↓
Forge
       ↓
Jira comment / Confluence macro
```

Forge Remoteは、自分でホストする外部サービスとForgeを統合するための仕組みです。ただし、外部処理を使うと「Runs on Atlassian」の適格性に影響する可能性があります。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/remote/?utm_source=chatgpt.com))

## Forge Automation Action

独自の処理をAutomationの「アクション」として追加することもできます。

たとえばAutomation画面に、

```text
Run AI PR Review
Generate Confluence Design Doc
Validate Evidence Bundle
Ask Bedrock Claude
Create Agent Workflow
```

という独自アクションを追加できます。

Atlassian自身も、標準Automationを超える複雑なロジックや外部システム統合にはForgeを利用する位置付けを示しています。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/building-automations/?utm_source=chatgpt.com))

---

# 6. Webhookとイベント駆動

APIで定期的にポーリングするより、Webhookを使う方が一般的です。

JiraではOAuth 2.0アプリなどからWebhookを動的登録できます。課題変更などを受けて外部処理を開始できます。([developer.atlassian.com](https://developer.atlassian.com/cloud/jira/platform/webhooks/?utm_source=chatgpt.com))

例：

```text
Jira issue updated
  ↓ webhook
EventBridge
  ↓
SQS
  ↓
Lambda
  ↓
Bedrock Claude
  ↓
Jira REST APIでコメント
  ↓
Confluenceにレビュー記録
```

実運用では、Webhook受信後にすぐAI処理を同期実行せず、SQSなどのキューへ入れることを勧めます。

理由は次の通りです。

- 重複イベントを排除できる
- リトライできる
- 順序を制御できる
- APIレート制限に対応できる
- AI処理が長くてもWebhookをすぐ応答できる
- 監査ログを残せる

JiraやConfluence CloudのAPIにはレート制限があるため、キャッシュ、バッチ処理、指数バックオフが必要です。([developer.atlassian.com](https://developer.atlassian.com/cloud/jira/platform/rate-limiting/?utm_source=chatgpt.com))

---

# 7. Atlassian Rovo MCP Server

現在、Atlassianは公式の **Atlassian Rovo MCP Server** を提供しています。

これはかなり重要です。

```text
Claude Code / Cursor / VS Code / AI Assistant
                 ↓ MCP
        Atlassian Rovo MCP Server
                 ↓
     Jira / Confluence / Compassなど
```

外部のAIアシスタントやIDEから、Jira・Confluenceなどのデータをリアルタイムに検索、作成、更新できます。([Atlassian](https://www.atlassian.com/platform/rovo-mcp?utm_source=chatgpt.com))

## できること

公式ドキュメント上、MCPクライアントから自然言語で次のような操作が可能です。

- Jira課題の検索
- Jira課題の取得
- Jira課題の作成
- Jira課題の更新
- コメントの読み書き
- Confluenceページの検索
- Confluenceページの取得
- Confluenceコンテンツの作成・更新
- Rovo検索
- 関連情報の横断検索
- Compass情報へのアクセス

利用可能なツールは、読み取り、書き込み、検索などの権限グループに整理され、組織管理者がグループ単位で許可・拒否できます。([Atlassian Support](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/supported-tools/?utm_source=chatgpt.com))

たとえばClaude Codeから次のように依頼できます。

```text
KAN-314の要件と関連するConfluence設計書を調べ、
現在のブランチの実装が要件を満たしているかレビューしてください。
重大な問題だけJiraにコメントしてください。
```

```text
このリポジトリに関連する未解決のJira課題を検索し、
今回の変更と競合する仕様がないか調べてください。
```

```text
今回の実装内容からConfluence用の設計変更案を作り、
既存ページを直接上書きせず、レビュー用の子ページを作成してください。
```

## 認証

主な認証はOAuth 2.1です。AtlassianはOAuth 2.1を第一の認証方式としています。([Atlassian Support](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/authentication-and-authorization/?utm_source=chatgpt.com))

公式MCPエンドポイントは、セットアップガイド上では次です。

```text
https://mcp.atlassian.com/v1/mcp/authv2
```

CursorやVS Code向けの公式セットアップ案内も提供されています。([Atlassian Support](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/getting-started-with-the-atlassian-remote-mcp-server/?utm_source=chatgpt.com))

組織管理者は、

- 接続を許可するクライアント
- 接続元ドメイン
- OAuth 2.1
- API Token認証の可否
- MCPの権限グループ

を制御できます。([Atlassian Support](https://support.atlassian.com/security-and-access-policies/docs/control-atlassian-rovo-mcp-server-settings/?utm_source=chatgpt.com))

## MCPの権限モデル

重要なのは、MCPがAtlassianの権限制御を迂回するわけではないことです。

基本的には、

```text
AIが見られる範囲
=
認証したユーザーが見られる範囲
∩
管理者がMCPに許可したツール
```

です。

したがって、ユーザーがアクセスできないJiraプロジェクトやConfluenceページを、MCP経由で突然見られるようになるわけではありません。公式も、既存権限、最小権限、高影響操作の確認、監査ログの監視を推奨しています。([Atlassian Support](https://support.atlassian.com/security-and-access-policies/docs/understand-atlassian-rovo-mcp-server/?utm_source=chatgpt.com))

## MCPを本番運用するときの注意

MCPは便利ですが、AIエージェントに無制限の書き込み権限を渡すべきではありません。

特に危険なのは次です。

```text
全プロジェクト検索
+ Jira一括更新
+ Confluence一括更新
+ 人間確認なし
```

安全な構成は次です。

```text
読み取り系MCP
  ├─ Jira search/read
  └─ Confluence search/read

書き込み
  ├─ Draftページ作成のみ
  ├─ 特定プロジェクトのみ
  ├─ 特定ラベルの課題のみ
  └─ 人間承認後に実行
```

AIレビューでは、いきなり課題をCloseしたり設計書を上書きしたりせず、

- コメントを下書きする
- ConfluenceにReview Draftを作る
- Jiraに`ai-proposed`ラベルを付ける
- 承認用Taskを作る

という構造が安全です。

---

# 8. Rovo、Rovo Agents、MCPの違い

混同しやすいため整理します。

## Rovo Search

Atlassian内外の組織知識を横断検索する機能。

## Rovo Chat

組織データに基づいて質問に答えるチャット。

## Rovo Agents

特定の目的を持つAtlassian内のAIエージェント。

## Rovo MCP Server

Claude Code、Cursor、ChatGPTなど、**外部AIクライアントからAtlassianデータへ接続する入口**。

RovoはSearch、Chat、Agentsを含むAI製品として提供されています。([Atlassian](https://www.atlassian.com/software/rovo?utm_source=chatgpt.com))

使い分けは次の通りです。

```text
Atlassianの画面内でAIを使う
→ Rovo Chat / Rovo Agents

Claude CodeやCursorからJira・Confluenceを使う
→ Rovo MCP Server

独自のUI・権限・処理を実装する
→ Forge / REST API / Forge Remote
```

---

# 9. 高度な連携の4段階

## レベル1：標準機能

```text
Jira
Confluence
標準マクロ
テンプレート
Smart Links
```

小規模チームや試験導入向けです。

## レベル2：Automation

```text
Trigger
Condition
Branch
Smart Values
Incoming Webhook
Web Request
```

一般的な業務自動化の多くはここで実現できます。

## レベル3：MCP

```text
Claude Code
Cursor
AI Agent
  ↓
Rovo MCP
  ↓
Jira / Confluence
```

AIがタスクと知識を参照しながら作業できます。

## レベル4：Forge＋独自バックエンド

```text
Forge UI
Forge Automation Action
Forge Web Trigger
Forge Queue
Forge Remote
AWS Bedrock
PostgreSQL
Vector DB
```

組織固有のワークフロー、AIレビュー、監査、独自UIを実装できます。

---

# 10. ユーザーのAI Agent・PRレビュー用途への適用

ユーザーが検討しているAI PRレビュー基盤では、次の構成がかなり相性がよいです。

```text
GitHub Pull Request
        ↓
GitHub App / Webhook
        ↓
Review Orchestrator
        ├─ diff取得
        ├─ repository context取得
        ├─ Context7で最新ライブラリ仕様取得
        ├─ DeepWikiでOSS仕様取得
        ├─ Jira MCPで要求・バグ・受入条件取得
        └─ Confluence MCPで設計書・ADR取得
        ↓
AWS Bedrock Claude
        ↓
Structured Review Result
        ├─ GitHub Review
        ├─ Jira Comment
        ├─ Jira Review Task
        └─ Confluence Evidence Page
```

この場合、JiraとConfluenceは次の役割に分けるのがよいです。

## Jiraに置くもの

- 要求
- バグ
- 受入条件
- 担当者
- 優先度
- ステータス
- Sprint
- リリース
- 依存関係
- PRとの対応関係
- AIレビューで発見された修正事項

## Confluenceに置くもの

- アーキテクチャ
- ADR
- API仕様
- コーディング規約
- セキュリティポリシー
- 過去の障害分析
- ライブラリ選定理由
- 実験記録
- AIレビューの根拠・Evidence Bundle
- プロジェクトの長期的知識

## GitHubに置くもの

- ソースコード
- Diff
- PR Discussion
- CI結果
- 実装に直接結びつくレビューコメント

---

# 11. Context Packとしての利用

Jira・ConfluenceをAIに丸ごと渡すのではなく、PRごとに **Context Pack** を生成するのが適切です。

```json
{
  "pullRequest": {
    "number": 314,
    "files": ["src/reviewer.ts"],
    "summary": "Add parallel review agents"
  },
  "jira": {
    "primaryIssue": "KAN-101",
    "acceptanceCriteria": [],
    "relatedIssues": [],
    "knownBugs": []
  },
  "confluence": {
    "architecturePages": [],
    "adrs": [],
    "codingPolicies": [],
    "securityPolicies": []
  },
  "externalKnowledge": {
    "context7": [],
    "deepwiki": []
  }
}
```

検索手順は次のようにすると精度が上がります。

```text
1. PR本文からJira Keyを抽出
2. Jira MCP/APIで課題・親Epic・リンク課題を取得
3. Jira課題のRemote LinkからConfluenceページを取得
4. Confluence検索で関連ADR・設計書を追加
5. 変更ファイル名・symbol名で追加検索
6. 更新日時と正本ラベルで絞る
7. 必要な節だけContext Packへ入れる
```

重要なのは、単なるベクトル類似度だけでなく、

- Jiraリンク
- ページ階層
- ラベル
- Space
- Component
- Fix Version
- Repository
- CODEOWNERS
- ADRのStatus
- 更新日
- Canonicalフラグ

をランキングに使うことです。

---

# 12. 推奨する実装構成

ユーザーの用途なら、次の3層構成が堅実です。

```text
第1層：Atlassian標準Automation
- 軽い通知
- ラベル付け
- ステータス更新
- Webhook起動
- 承認フロー

第2層：Rovo MCP
- Claude Codeからの検索
- Jira要件取得
- Confluence設計取得
- 人間指示による軽い書き込み

第3層：独自Orchestrator
- Bedrock Claude
- 複数レビューAgent
- Context Pack生成
- Context7 / DeepWiki
- キャッシュ
- キュー
- 監査
- Evidence管理
```

ConfluenceにはForge Macroを作り、AIレビューの状態を表示します。

```text
AI Review Status
────────────────────────────
PR                 #314
Jira               KAN-101
Context freshness  97%
Agents completed   5 / 5
Critical findings  1
Warnings           4
Evidence           18 sources
Approval            Pending
```

JiraにはForge Issue Panelを作ります。

```text
Implementation Evidence
────────────────────────────
PR                  #314
Design document     Architecture v3
Acceptance criteria 7 / 8 satisfied
Open blockers       1
Review confidence   High
```

同じForgeアプリでConfluence MacroとJira Issue Panelを提供できるため、両画面のデータモデルを統一できます。([developer.atlassian.com](https://developer.atlassian.com/platform/forge/build-an-app-compatible-with-confluence-and-jira/?utm_source=chatgpt.com))

---

# 13. 全体評価

Atlassianの強みは、単にJiraとConfluenceが高機能なことではなく、次の拡張レイヤーが連続している点です。

```text
標準UI
  ↓
Automation
  ↓
Webhook / REST API
  ↓
MCP
  ↓
Forge UI
  ↓
Forge Backend
  ↓
Forge Remote / 独自クラウド
```

特に現在は公式Rovo MCP Serverがあるため、Claude CodeやCursorなどのAIエージェントに、

- 現在の要求
- 過去の設計判断
- 既知のバグ
- プロジェクトの進捗
- 関連する議論

を直接取得させられます。

ただし、**MCPだけで完全な自動化基盤を作るのは適切ではありません**。MCPは対話的な検索・操作には非常に強い一方で、大量処理、再試行、冪等性、監査、厳密なContext Pack生成には独自Orchestratorの方が適しています。

したがって最適な位置付けは、

> **Jiraを構造化された業務状態、Confluenceを正本知識、Rovo MCPをAIの対話的アクセス層、ForgeをUI拡張層、独自バックエンドを高度な実行・監査層として使う**

という構成です。

---

# you asked

message time: 2026-07-27 21:48:25

これが1番気になるところです。
どの方法を使うにしてもAI手動でconfluenceを作成、編集させる上での1番の課題が、特殊なデータ形式です。マークダウンでも単純なリッチテキストでもなく、非常に使いづらいです。なぜこのような形式になっているのか、どのような構造なのかの詳細を説明し、いい変換ツールや推奨される方法がないか調べてください

---

# chatgpt response

Confluence本文の「保存形式・API形式・エディタ内部形式」を切り分け、なぜ複雑なのか、AI編集で壊れやすい点、実用的な変換ライブラリと推奨設計まで確認します。

# 結論

AIにConfluenceを作成・編集させる場合、**AIへADFやConfluence Storage Formatを直接生成させる設計は避けるべきです**。

最も安定する構成は次です。

```text
AIが生成・編集する形式
    Markdown
       ↓
Markdown AST
       ↓
検証済み変換器
       ↓
Confluence Storage Format
       ↓
Confluence REST API
```

既存ページを部分編集するときは、

```text
Confluence Storage Format
       ↓
内部の文書AST
       ↓
対象セクションだけ置換
       ↓
Storage Formatへ再生成
       ↓
versionを確認して更新
```

とします。

現状、Confluence Cloudでは**Storage Formatが最も実用的な書き込み対象**です。ADFは構造的には優れていますが、Confluence特有のマクロ、画像、Smart Link、拡張機能まで扱うAI自動編集の中間形式としては、まだ扱いにくい部分があります。

---

# 1. なぜMarkdownではないのか

Markdownは人間が文章を書くには優れていますが、Confluenceが扱うものは単なる文章ではありません。

Confluenceページには、次のような要素があります。

- 段落、見出し、リスト、表
- メンション
- Jira課題カード
- Smart Link
- 添付画像
- ページ内コメント
- 展開パネル
- ステータス
- 日付
- タスク
- Confluenceマクロ
- Marketplaceアプリのマクロ
- レイアウト、列
- ページ参照
- ユーザー参照
- 埋め込みコンテンツ

たとえばMarkdownでは、

```markdown
[KAN-101](https://example.atlassian.net/browse/KAN-101)
```

は単なるリンクです。

しかしConfluenceでは、同じものを「Jira課題カード」として表示し、

- 課題キー
- タイトル
- ステータス
- 担当者
- 優先度

を動的に表示できます。

これは文字列だけでは表現できません。内部的には、「この部分はURL文字列ではなく、AtlassianのJiraオブジェクトを参照するノードである」という構造が必要です。

同様に、次のようなConfluenceマクロはMarkdownには標準表現がありません。

```text
情報パネル
展開可能セクション
目次
Jira課題一覧
ページプロパティ
Mermaidマクロ
draw.ioマクロ
Marketplaceアプリの独自マクロ
```

そのためConfluenceでは、単純なMarkdownやHTMLではなく、**構造化された文書モデルと製品固有の拡張表現**が必要になります。

---

# 2. 実際には複数の形式が重なっている

Confluenceの難しさは、「本文形式」が一つではないことです。

大きく分けると次の4層があります。

```text
1. エディタ内部の文書モデル
2. 保存・API書き込み形式
3. 表示用HTML
4. エクスポート用HTML
```

REST APIでは、主に次のrepresentation名が現れます。

| representation | 役割 |
|---|---|
| `storage` | Confluence固有のXHTML系保存形式 |
| `atlas_doc_format` | ADFのJSON文書 |
| `editor` | エディタ向け内部表現 |
| `view` | 通常表示用にレンダリングされたHTML |
| `styled_view` | スタイルを含む表示表現 |
| `export_view` | PDF・Word等の出力に近いHTML |

Atlassianの変換APIでは、現在、`storage`と`atlas_doc_format`から各表示形式への変換や、両者間の変換などが提供されています。ただし、Markdownは公式変換元・変換先に含まれていません。([Atlassian Developer](https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content-body/))

つまり、

```text
Markdown → Confluence
```

は公式APIだけでは完結せず、自前または外部ライブラリの変換器が必要です。

---

# 3. Storage Formatとは何か

## 基本はXHTML

Confluence Storage Formatは、HTMLに似ていますが、実際には**XMLとして正しいXHTML＋Confluence固有名前空間**です。

単純な文章は次のようになります。

```xml
<h1>設計概要</h1>
<p>
  このシステムは
  <strong>イベント駆動</strong>
  で動作します。
</p>

<ul>
  <li><p>GitHub Webhookを受信する</p></li>
  <li><p>Jira課題を検索する</p></li>
</ul>
```

ただし、通常のHTMLより厳格です。

たとえばHTMLでは許容されることがある、

```html
<br>
<img src="image.png">
```

はXMLとしては閉じていないため問題になります。

Storage Formatでは、原則として次のようにします。

```xml
<br />
<img src="image.png" />
```

タグの入れ子も正しくなければなりません。

```xml
<!-- 不正 -->
<strong><em>text</strong></em>

<!-- 正しい -->
<strong><em>text</em></strong>
```

MarkdownからStorage Formatへ変換する`md2conf`も、Storage Formatを「Confluence固有タグを追加したXHTML派生形式」と説明し、XMLとして不正なタグや不正な入れ子はエラーになるとしています。([GitHub](https://github.com/hunyadi/md2conf))

## Confluence固有タグ

複雑な要素では、`ac:`や`ri:`という名前空間付きタグが使われます。

- `ac:`：Atlassian Confluenceの構造・マクロ
- `ri:`：Resource Identifier、ページ・添付・ユーザーなどの参照

### コードブロック

```xml
<ac:structured-macro ac:name="code">
  <ac:parameter ac:name="language">python</ac:parameter>
  <ac:plain-text-body><![CDATA[
def hello():
    print("Hello")
  ]]></ac:plain-text-body>
</ac:structured-macro>
```

### 情報パネル

```xml
<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p>この処理は本番環境では実行しないでください。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

### 展開可能セクション

```xml
<ac:structured-macro ac:name="expand">
  <ac:parameter ac:name="title">詳細を表示</ac:parameter>
  <ac:rich-text-body>
    <p>ここに詳細情報があります。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

### 添付画像

```xml
<ac:image>
  <ri:attachment ri:filename="architecture.png" />
</ac:image>
```

### Confluenceページへのリンク

```xml
<ac:link>
  <ri:page ri:content-title="システム設計書" />
  <ac:link-body>設計書を参照</ac:link-body>
</ac:link>
```

### ユーザーメンション

```xml
<ac:link>
  <ri:user ri:account-id="ACCOUNT_ID" />
</ac:link>
```

ここが、AIに直接書かせると難しくなる主要因です。

AIはXMLとして見た目が正しいものを生成しても、

- マクロ名が正しくない
- パラメータ名が違う
- `CDATA`が壊れる
- 添付ファイルが実際には存在しない
- ページ参照名が曖昧
- Marketplaceアプリ固有のmacro IDを捏造する
- 名前空間の使い分けを誤る

といった問題を起こします。

---

# 4. ADFとは何か

ADFは **Atlassian Document Format** の略です。

JSONによる木構造で、考え方としてはProseMirror系の文書モデルに近いものです。

最小構造は次のようになります。

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [
        {
          "type": "text",
          "text": "Hello "
        },
        {
          "type": "text",
          "text": "world",
          "marks": [
            {
              "type": "strong"
            }
          ]
        }
      ]
    }
  ]
}
```

ADFは、

- `doc`
- `paragraph`
- `heading`
- `bulletList`
- `orderedList`
- `listItem`
- `table`
- `tableRow`
- `tableCell`
- `codeBlock`
- `panel`
- `media`
- `mention`
- `inlineCard`

などの**ノード**で構成されます。

太字、リンク、文字色などは、テキストノードに付ける**mark**として表現されます。Atlassianの公式説明でも、ADFはブロックノードとインラインノードの階層で構成され、装飾はmarkとして表現されるとされています。([Atlassian Developer](https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/))

## 見出し

```json
{
  "type": "heading",
  "attrs": {
    "level": 2
  },
  "content": [
    {
      "type": "text",
      "text": "設計概要"
    }
  ]
}
```

## リンク

```json
{
  "type": "text",
  "text": "Atlassian",
  "marks": [
    {
      "type": "link",
      "attrs": {
        "href": "https://www.atlassian.com"
      }
    }
  ]
}
```

## 表

```json
{
  "type": "table",
  "content": [
    {
      "type": "tableRow",
      "content": [
        {
          "type": "tableHeader",
          "content": [
            {
              "type": "paragraph",
              "content": [
                {
                  "type": "text",
                  "text": "項目"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

## ADFが作られた理由

ADFのメリットは明確です。

### 構造を保持できる

MarkdownやHTML文字列ではなく、見出し、リスト、表、メンション、カードなどを型付きノードとして扱えます。

### エディタで安全に操作できる

たとえば太字の一部分だけを変更したり、表のセルを追加したり、ノード単位で操作できます。

### 製品間で共通化できる

Jiraの説明・コメント、Confluence、その他Atlassian製品で同じような文書部品を共有できます。

### 不正HTMLを排除できる

任意のHTMLやJavaScriptではなく、許可されたノードと属性だけに制限できます。

### リアルタイム共同編集に向く

文字列ではなく文書ツリーとして扱えるため、共同編集エンジンとの相性がよくなります。

---

# 5. それでもADFがAI編集に向かない理由

ADFはコンピューターにとっては合理的ですが、LLMに直接全文生成させるには冗長です。

たとえば、

```markdown
## 結論

**Storage Format**を使う。
```

は数十文字ですが、ADFでは次のようになります。

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": {
        "level": 2
      },
      "content": [
        {
          "type": "text",
          "text": "結論"
        }
      ]
    },
    {
      "type": "paragraph",
      "content": [
        {
          "type": "text",
          "text": "Storage Format",
          "marks": [
            {
              "type": "strong"
            }
          ]
        },
        {
          "type": "text",
          "text": "を使う。"
        }
      ]
    }
  ]
}
```

問題は次の通りです。

- トークン消費量が大きい
- JSONの括弧や配列を壊しやすい
- ノードの親子関係を誤りやすい
- 製品ごとに対応ノードが異なることがある
- スキーマに存在しても対象APIで使えないノードがある
- 画像やメンションには外部IDが必要
- マクロや拡張ノードの互換性が分かりにくい

公式ドキュメントも、ADFのJSON Schemaに含まれるすべてのノード・markが、各実装で利用できるとは限らないと注意しています。([Atlassian Developer](https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/))

そのため、AIにはADF全体を生成させず、

```json
{
  "operation": "replace_section",
  "heading": "設計概要",
  "markdown": "## 設計概要\n\n..."
}
```

のような、より小さい編集命令を出させる方が安全です。

---

# 6. `editor`、`view`、`export_view`は何か

## `editor`

エディタで扱うための内部表現です。

これは公開API利用者が正本として長期保存する形式には向きません。Atlassian内部のエディタ実装や移行に左右されやすいため、通常は`storage`または`atlas_doc_format`を利用します。

## `view`

ユーザーがページを閲覧するときに近い、レンダリング済みHTMLです。

```html
<h1>設計概要</h1>
<p>このページは...</p>
<div class="confluence-information-macro">...</div>
```

読み取り、検索インデックス、スクレイピング、LLMへの入力には便利ですが、**これを編集して書き戻してはいけません**。

理由は、

- 表示用クラスが混ざる
- 内部IDが混ざる
- マクロが展開済みになる
- 元のマクロパラメータを復元できない
- 情報が不可逆に失われる

ためです。

## `styled_view`

`view`にさらに表示用スタイル情報が付いたものです。

主にレンダリング用途であり、編集用ではありません。

## `export_view`

PDF、Word、HTMLエクスポートに近い形式です。

これも見た目を再現するための形式であり、編集用ではありません。

したがって、用途は次のように整理できます。

```text
書き込み・編集
→ storage または atlas_doc_format

閲覧・LLMへの読み込み
→ view または storageから抽出したMarkdown

外部出力
→ export_view

エディタ内部処理
→ editor
```

---

# 7. Confluence Cloudでは何を送るべきか

REST API v2のページ作成例では、公式サンプルが次の形式を示しています。

```json
{
  "spaceId": "123456",
  "status": "current",
  "title": "設計書",
  "parentId": "654321",
  "body": {
    "representation": "storage",
    "value": "<h1>設計概要</h1><p>...</p>"
  }
}
```

Confluence REST API v2のページ作成APIでも、公式サンプルは`representation: "storage"`を使用しています。([Atlassian Developer](https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/))

既存ページの取得では、

```http
GET /wiki/api/v2/pages/{pageId}?body-format=storage
```

または、

```http
GET /wiki/api/v2/pages/{pageId}?body-format=atlas_doc_format
```

のように本文形式を指定します。

更新時は通常、現在のversion番号を取得し、1増やして送信します。

概念的には次のようになります。

```json
{
  "id": "123456",
  "status": "current",
  "title": "設計書",
  "body": {
    "representation": "storage",
    "value": "<h1>更新後</h1><p>...</p>"
  },
  "version": {
    "number": 8,
    "message": "AIによる設計概要の更新"
  }
}
```

同時に人間が編集していた場合、古いversionのまま更新すると競合します。

---

# 8. 推奨変換ツール

## 第一候補：`md2conf`

Python系で最も実用的な候補です。

GitHub上の`hunyadi/md2conf`、PyPI名は`markdown-to-confluence`です。

```bash
pip install markdown-to-confluence
```

このツールは、

```text
Markdown
  ↓
Confluence Storage Format
  ↓
REST API
```

を一通り処理します。

対応機能がかなり充実しています。

- 見出し
- 太字、斜体、取り消し線
- リスト
- 表
- コードブロック
- 画像アップロード
- 添付ファイル
- 脚注
- GitHub Alerts
- info、warning等のパネル
- 折りたたみ
- タスクリスト
- Mermaid
- PlantUML
- draw.io
- LaTeX数式
- Confluence Status
- 日付ウィジェット
- ページ階層
- ローカルでStorage Formatだけを出力

といった機能があります。MarkdownをStorage Formatへ変換し、画像や本文をAPIでアップロードすることを目的としたツールです。([GitHub](https://github.com/hunyadi/md2conf))

### 適しているケース

- MarkdownをGitで管理する
- GitHub ActionsからConfluenceへ同期する
- AIにMarkdownを書かせる
- 技術文書を自動公開する
- MermaidやPlantUMLを使う
- PythonベースのAgentを作る

### 注意点

基本的には**一方向同期**に向いています。

```text
Markdown正本 → Confluenceミラー
```

として使うと強いですが、

```text
人間がConfluenceで編集
＋
AIがMarkdownから編集
```

を同じページで混在させると、どちらが正本か分からなくなります。

---

## 第二候補：自前のMarkdown AST → Storage変換

長期的には、これが最も堅牢です。

Node.jsなら、

```text
remark-parse
remark-gfm
unist
```

などでMarkdownをASTに変換し、自分でStorage Formatへシリアライズします。

概念的には、

```typescript
type DocumentNode =
  | { type: "heading"; level: number; children: InlineNode[] }
  | { type: "paragraph"; children: InlineNode[] }
  | { type: "bulletList"; items: DocumentNode[][] }
  | { type: "codeBlock"; language?: string; code: string }
  | { type: "panel"; panelType: "info" | "warning"; children: DocumentNode[] }
  | { type: "confluenceMacro"; name: string; parameters: Record<string, string> };
```

という内部ASTを作ります。

変換パイプラインは、

```text
Markdown
  ↓ remark
MDAST
  ↓ 独自normalize
Canonical Document AST
  ↓ Confluence renderer
Storage Format
```

とします。

### 利点

- AIが生成する形式をMarkdownに限定できる
- Confluence依存部分を一カ所に隔離できる
- マクロを型安全に扱える
- XML検証を入れられる
- 将来ADFへ切り替えやすい
- テストしやすい

### 特に重要な設計

AIに直接、

```xml
<ac:structured-macro ...>
```

を書かせるのではなく、次のような独自記法を書かせます。

```markdown
:::confluence-panel{type="warning"}
本番データを削除しないでください。
:::
```

または、

```markdown
```confluence-macro
{
  "name": "jira",
  "parameters": {
    "jql": "project = KAN AND status != Done"
  }
}
```
```

これを変換器側で安全なStorage Formatへ変換します。

---

## 第三候補：AtlaskitのMarkdown Transformer

Atlassianのフロントエンドライブラリ群には、

```text
@atlaskit/editor-markdown-transformer
@atlaskit/adf-schema
@atlaskit/editor-json-transformer
```

などがあります。

Markdown Transformerを利用してMarkdownからADFに近い文書モデルへ変換する例は、Atlassian Developer Communityでも案内されています。([The Atlassian Developer Community](https://community.developer.atlassian.com/t/render-markdown-document-as-body-of-confluence-page-using-forge-app/47592?utm_source=chatgpt.com))

ただし、これを中心に据える場合は注意が必要です。

- パッケージ間の依存関係が大きい
- Atlaskit内部APIに近い部分がある
- Node.jsバックエンドで使いにくいことがある
- Confluence固有マクロを完全には表現しにくい
- Jira向けADFとConfluence向けADFの差がある
- バージョン更新で挙動が変わる可能性がある

そのため、

```text
React/Forge/Atlassian UI内でADFを扱う
```

には向きますが、

```text
汎用AI Agentの安定した文書変換基盤
```

としては、Storage Format変換器の方が扱いやすいケースが多いです。

---

## 第四候補：`marklassian`

`marklassian`はMarkdownからADFへ変換する軽量JavaScriptライブラリです。2026年3月時点でnpmパッケージとして公開されています。([npm](https://npmjs.com/package/marklassian?utm_source=chatgpt.com))

用途としては、

```text
Markdown → Jiraの説明・コメント用ADF
```

に特に分かりやすい候補です。

ただしConfluenceについては、

- ページAPIが受け入れるADFの範囲
- マクロ
- 画像・添付
- Smart Link
- Marketplace拡張
- Confluence固有ノード

を別途検証する必要があります。

したがって、これは**Jira用ADF変換器としては有力**ですが、Confluenceの複雑なページ全体を扱う第一候補とはまだ言いにくいです。

---

# 9. 公式の変換APIは使えるか

Atlassianは次のような変換を提供しています。

```text
storage → atlas_doc_format
storage → view
storage → export_view
atlas_doc_format → storage
atlas_doc_format → view
editor → storage
```

ただしこれは非同期APIです。

```http
POST /wiki/rest/api/contentbody/convert/async/storage
```

のようにリクエストし、`asyncId`を取得して、結果を別APIからポーリングします。変換結果は完了後5分間取得可能です。([Atlassian Developer](https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-content-body/))

### 有効な用途

- ADFからStorage Formatへ変換する
- Storage Formatを表示用HTMLへ変換する
- 自前のADF生成結果を公式変換で検証する
- Confluenceのコンテキストを考慮してマクロをレンダリングする

### 解決できないこと

- Markdown → Storage
- Markdown → ADF
- 自然言語 → 正しいマクロ
- 既存ページの意味的な差分編集
- Marketplaceマクロの安全な生成

したがって、公式変換APIは**補助的な変換・検証層**として使うのがよく、Markdown変換器の代替ではありません。

---

# 10. AIによる新規ページ作成の推奨方法

新規作成は比較的簡単です。

## 推奨フロー

```text
1. AIがMarkdownを生成
2. MarkdownパーサーでAST化
3. 許可要素だけに正規化
4. Storage Formatへ変換
5. XMLとして検証
6. APIでdraftページを作成
7. 再取得して表示確認
8. 人間が承認
9. currentとして公開
```

AI出力例：

```markdown
# PRレビュー基盤

## 目的

PRに関連する要件と設計を自動収集する。

## コンポーネント

| コンポーネント | 役割 |
|---|---|
| GitHub App | Webhook受信 |
| Context Builder | 文脈収集 |
| Reviewer | コードレビュー |

:::panel{type="warning"}
Jiraの自動更新は人間の承認後に実行する。
:::
```

変換器が生成するStorage Format：

```xml
<h1>PRレビュー基盤</h1>
<h2>目的</h2>
<p>PRに関連する要件と設計を自動収集する。</p>

<h2>コンポーネント</h2>
<table>
  ...
</table>

<ac:structured-macro ac:name="warning">
  <ac:rich-text-body>
    <p>Jiraの自動更新は人間の承認後に実行する。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

---

# 11. 既存ページ編集が最も難しい

新規作成より、既存ページの部分編集の方がはるかに難しいです。

悪い方法は次です。

```text
既存ページ全体をMarkdown化
  ↓
AIに全文を書き直させる
  ↓
Storage Formatに戻す
  ↓
全面上書き
```

この方法では、

- マクロが消える
- inline commentの位置が壊れる
- Smart Linkが普通のリンクになる
- 添付画像の参照が変わる
- ページ内アンカーが壊れる
- Marketplaceマクロが失われる
- 人間が同時に行った変更を消す
- 差分が巨大になる

という問題が起きます。

## 推奨：セクション単位編集

見出し単位で文書を分解します。

```text
# タイトル

## 背景
...

## 設計
...

## リスク
...
```

AIには次のような操作だけを返させます。

```json
{
  "baseVersion": 17,
  "operations": [
    {
      "type": "replace_section",
      "headingPath": ["設計"],
      "contentMarkdown": "..."
    },
    {
      "type": "append_section",
      "headingPath": ["リスク"],
      "contentMarkdown": "..."
    }
  ]
}
```

実際のStorage Format操作は、決定論的なプログラムが行います。

---

# 12. さらに安全な方法：管理ブロックだけを編集する

人間編集領域とAI編集領域を分けます。

Markdown側では、

```markdown
<!-- AI-MANAGED:START architecture-summary -->
## アーキテクチャ概要

...
<!-- AI-MANAGED:END architecture-summary -->
```

Storage Formatでは、たとえば拡張マクロや専用マクロ、識別可能なマーカーで囲みます。

```xml
<p data-ai-block-id="architecture-summary">
  ...
</p>
```

ただし、未知のHTML属性がConfluence側で保持される保証は弱いため、より確実なのは、

- 専用Forge Macro
- Content Property
- ページ内の固定見出し
- AI管理専用子ページ
- AI-generatedラベル

を使うことです。

最も安全なのは次の構成です。

```text
人間管理ページ
├─ 背景
├─ 設計方針
└─ AI生成結果マクロ
       ↓
    外部DBまたはForge Storage
```

AIは本文全体を触らず、マクロが表示するデータだけを更新します。

---

# 13. マクロをAIに編集させる推奨方法

AIにマクロXMLを直接作らせないでください。

代わりに、型付きオブジェクトを返させます。

```json
{
  "type": "confluence_macro",
  "macro": "jira",
  "parameters": {
    "jql": "project = KAN AND labels = ai-review",
    "columns": ["key", "summary", "status", "assignee"],
    "maximumIssues": 20
  }
}
```

そしてコード側で、

- 許可されたマクロ名か
- 許可されたパラメータか
- JQLが危険でないか
- 最大件数を超えていないか
- 対象プロジェクトが許可範囲か

を検証し、Storage Formatへ変換します。

```typescript
const allowedMacros = {
  jira: {
    parameters: ["jql", "columns", "maximumIssues"],
  },
  info: {
    parameters: ["title"],
  },
  expand: {
    parameters: ["title"],
  },
} as const;
```

Marketplaceマクロについては、人間がGUIで正しいマクロを一度作り、そのStorage Formatを取得して、変更可能なパラメータだけをテンプレート化する方法が安全です。

`md2conf`のドキュメントでも、Marketplaceアプリ統合では、まずGUIでウィジェットを作成し、Storage Formatを確認して構造を調べる方法が案内されています。([GitHub](https://github.com/hunyadi/md2conf))

---

# 14. 読み取り時はMarkdownへ落としてよい

AIに既存ページを理解させるだけなら、Storage Formatそのものを渡す必要はありません。

推奨される読み取りパイプラインは、

```text
Storage Format
    ↓
XML Parser
    ↓
Confluence AST
    ↓
LLM向けMarkdown
```

です。

変換時に、マクロを単に消すのではなく、意味を残します。

元のStorage Format：

```xml
<ac:structured-macro ac:name="jira">
  <ac:parameter ac:name="jql">
    project = KAN AND status != Done
  </ac:parameter>
</ac:structured-macro>
```

LLM向けMarkdown：

```markdown
:::confluence-macro{type="jira"}
JQL: project = KAN AND status != Done
:::
```

画像も、

```markdown
![architecture.png](confluence-attachment://architecture.png)
```

のように参照情報を残します。

メンションも、

```markdown
@山田太郎{accountId="..."}
```

のように保持します。

つまり、**LLM用Markdownは通常のMarkdownより少し拡張した可逆形式**にするのが理想です。

---

# 15. 一番おすすめのアーキテクチャ

ユーザーのAI Agent基盤なら、次のようにします。

```text
                    ┌────────────────────┐
                    │ Canonical Document │
                    │       AST          │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
        Markdown Renderer  Confluence       ADF Renderer
              │           Storage Renderer       │
              │               │                │
          Git / AI       Confluence API      Jira API
```

## Canonical AST

```typescript
interface Document {
  version: 1;
  blocks: Block[];
}

type Block =
  | Heading
  | Paragraph
  | List
  | Table
  | CodeBlock
  | Panel
  | Image
  | ConfluenceMacro
  | JiraCard
  | UnknownConfluenceNode;
```

重要なのは、未知のConfluence要素を消さないことです。

```typescript
interface UnknownConfluenceNode {
  type: "unknown_confluence_node";
  originalStorageXml: string;
  readOnly: true;
}
```

AIが理解できないマクロがあっても、`originalStorageXml`をそのまま保持し、編集対象外として書き戻します。

---

# 16. 用途別の最適解

| 用途 | 推奨形式・方法 |
|---|---|
| AIによる新規技術文書作成 | Markdown → `md2conf` → Storage |
| Git管理の文書をConfluenceへ同期 | Markdownを正本にして`md2conf` |
| Jira説明・コメント生成 | Markdown → ADF変換 |
| Confluenceの複雑なマクロ入りページ | Storage Format＋型付きテンプレート |
| 既存ページの一部更新 | AST化して見出し単位パッチ |
| AIが作るレポート表示 | Forge Macroで外部データを表示 |
| 人間とAIが同じページを編集 | AI管理セクションまたは専用子ページ |
| ページ全体を双方向同期 | 独自Canonical ASTが必要 |
| 単なる読み取り・要約 | Storage/View → LLM向けMarkdown |
| Marketplaceマクロ | GUIで作ったStorageをテンプレート化 |

---

# 最終的な推奨

最初の実装としては、次の構成が最も現実的です。

```text
AI
 ↓
拡張Markdown
 ↓
md2confまたは独自Markdown AST変換器
 ↓
Confluence Storage Format
 ↓
XML Schema・許可マクロ検証
 ↓
Draftページ作成
 ↓
人間承認
```

既存ページの編集は、

```text
既存Storage Format取得
 ↓
XML AST化
 ↓
見出し単位で分割
 ↓
AIはJSON Patch風の編集命令だけ返す
 ↓
プログラムが対象部分だけ置換
 ↓
未知マクロは元XMLを完全保持
 ↓
version競合確認
 ↓
更新
```

とするのがよいです。

最重要ポイントは、**MarkdownをConfluenceの保存形式にしようとせず、AIと人間が扱う編集言語として使い、Confluence固有形式は変換アダプターの内側に隔離すること**です。これなら将来Confluence APIやADFの仕様が変わっても、AI Agent本体や文書生成プロンプトを大きく変更せずに済みます。

---

# you asked

message time: 2026-07-28 11:30:20

私が社内でconfluenceのapiを使えるかわかりません。多分無理だと思われます。その場合に、どのようにして文章を追加、編集するのがベストでしょうか

---

# chatgpt response

APIが使えない場合は、権限状況に応じて「貼り付け補助」「ブラウザ自動操作」「公式自動化」の三段階になります。Confluence Cloudの現行エディタで壊れにくい方法を確認します。

# 結論

社内でConfluence APIを使えない場合、現実的に最も安定するのは次の方法です。

```text
AIがMarkdownで文章を作る
        ↓
ローカルで「コピー可能なリッチテキスト」に変換
        ↓
Confluenceの編集画面へ貼り付け
        ↓
マクロ・画像・リンクだけ人間が調整
        ↓
差分を確認して公開
```

**Storage FormatやADFを経由する必要はありません。**
ブラウザのクリップボードに、プレーンテキストではなく`text/html`形式のリッチテキストを入れれば、Confluenceのエディタが見出し、太字、リスト、リンク、単純な表などを自分の内部形式へ変換して取り込みます。Atlassianも、基本的な書式を持つテキストはエディタへ直接貼り付けられると案内しています。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/import-content-into-confluence-cloud/?utm_source=chatgpt.com))

新規ページ全体なら、**WordまたはGoogle Docs経由のインポート**も有力です。Confluence CloudはWord文書、Google Docs、OneDrive文書のインポートを公式に提供しています。([Atlassian Support](https://support.atlassian.com/ja/confluence-cloud/docs/import-content-into-confluence-cloud/?utm_source=chatgpt.com))

---

# 1. 最初に確認すべき権限

「APIが使えない」という状態には、いくつかの種類があります。

| 状況 | 可能な方法 |
|---|---|
| APIトークン発行だけ禁止 | 手動貼り付け、Wordインポート、ブラウザ補助 |
| OAuthアプリ接続禁止 | 同上 |
| Marketplaceアプリ追加禁止 | ローカル変換＋貼り付け |
| ブラウザ拡張機能も禁止 | コピー＆ペースト、Wordインポート |
| 自動操作ツール禁止 | 完全手動の貼り付け |
| Confluence Automationを使える | テンプレート、定型ページ作成など |
| Rovoを使える | Confluence内での生成・要約・編集補助 |
| ページ編集権限しかない | リッチテキスト貼り付けが中心 |

APIが禁止されていても、通常の編集画面でページを作成・更新できるなら、**クリップボード経由の入力は可能**です。

ただし、ブラウザ自動操作や拡張機能は、技術的に可能でも社内規則上は禁止されている可能性があります。特に認証済みConfluenceのページ内容を外部AIへ送信する場合は、APIの可否よりも、**社内情報を外部サービスへ入力してよいか**の確認が重要です。

---

# 2. 一番おすすめ：リッチテキストとしてコピー＆ペースト

## Markdownをそのまま貼る方法は不安定

Confluence CloudはMarkdown入力ショートカットをサポートしています。

たとえば行頭で、

```markdown
# 見出し
* 箇条書き
> 引用
```

などを入力すると、対応するリッチテキスト要素へ変換されます。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/available-markdown-commands/?utm_source=chatgpt.com))

ただし、これは主に**入力中のショートカット**です。長いMarkdown文書をそのまま一括貼り付けたときに、すべてが完全に変換されることを前提にするべきではありません。

たとえば次は失敗しやすい部分です。

- 複雑な表
- 入れ子リスト
- コードブロック
- Markdown内のHTML
- 脚注
- Mermaid
- 数式
- Callout記法
- タスクリスト
- ローカル画像
- 相対リンク

したがって、AIが作ったMarkdownを一度HTMLへ変換し、**レンダリング結果をコピー**する方が安定します。

## 推奨フロー

```text
ChatGPT / Claude
    ↓ Markdown
ローカルのプレビュー画面
    ↓ Copy rich text
Confluence編集画面
    ↓ Paste
公開前に確認
```

HTMLとして保持しやすい要素は次です。

- 見出し
- 段落
- 太字
- 斜体
- リンク
- 箇条書き
- 番号付きリスト
- 引用
- 単純な表
- インラインコード
- 水平線

一方、次はConfluence上で後から設定するのがよいです。

- Info・Warningなどのパネル
- Expand
- Jiraマクロ
- 目次
- Status
- Smart Link
- ユーザーメンション
- 添付ファイル
- Mermaid・draw.io
- Marketplaceアプリのマクロ
- 複雑なレイアウト

---

# 3. 最も簡単な実践方法

## 方法A：ChatGPTの表示結果を直接コピー

AIに、次のように指示します。

```text
Confluenceへ貼り付けるため、通常のMarkdownコードブロックではなく、
画面上でレンダリングされる文書として出力してください。

使用してよい要素:
- 見出し1〜3
- 段落
- 太字
- 箇条書き
- 番号付きリスト
- 単純な表
- 引用
- コードブロック

Confluence固有マクロが必要な場所には、
[CONFLUENCE: INFO PANEL]
[CONFLUENCE: JIRA MACRO]
のような目印を入れてください。
```

そのレンダリング結果を選択してコピーし、Confluenceへ貼り付けます。

これは最小構成ですが、次の問題があります。

- ChatGPT画面からのコピー方法によってはプレーンテキストになる
- 大きな表が崩れることがある
- コードブロックが通常段落になることがある
- 見出し階層が完全には維持されないことがある

それでも、普通の文章中心なら十分実用的です。

## 方法B：Google DocsまたはWordを中継する

```text
AIの文章
  ↓
Google Docs / Wordへ貼り付け
  ↓
書式を確認
  ↓
Confluenceへ貼り付け
```

または、新規ページ全体ならConfluenceの公式インポート機能を使います。

```text
AI
 ↓
.docx生成
 ↓
Confluence
 ↓
テンプレートとインポート
 ↓
Word文書を選択
```

Confluence Cloudは、ページ作成画面の「テンプレートとインポート」からWord、Google Docs、OneDriveを取り込めます。([Atlassian Support](https://support.atlassian.com/ja/confluence-cloud/docs/import-content-into-confluence-cloud/?utm_source=chatgpt.com))

### Wordインポートが向く場合

- 新しい長文ページを作る
- 表が多い
- 数十ページ相当の文章
- 見出し階層を維持したい
- 画像を含む
- 一度だけ取り込めればよい

### 向かない場合

- 既存ページの一部だけ更新する
- 頻繁に同期する
- Confluenceマクロを多用する
- 人間の編集済み部分を保持したい

Wordインポートは「新規ページ作成」には強いですが、「既存ページへの差分適用」には向きません。

---

# 4. ローカルの「Confluence貼り付け用コンバーター」を作る

社内でAPIが使えないなら、最終的にはこれが最も便利です。

小さなローカルWebアプリまたはCLIを作り、

```text
左側：Markdown
右側：Confluence貼り付けプレビュー
ボタン：リッチテキストとしてコピー
```

という構成にします。

## 内部処理

```text
Markdown
  ↓
Markdown AST
  ↓
安全なHTMLへ変換
  ↓
navigator.clipboard.write()
  ├─ text/html
  └─ text/plain
```

クリップボードに2種類を同時に入れます。

```javascript
const clipboardItem = new ClipboardItem({
  "text/html": new Blob([html], { type: "text/html" }),
  "text/plain": new Blob([plainText], { type: "text/plain" }),
});

await navigator.clipboard.write([clipboardItem]);
```

Confluence側が`text/html`を受け取ればリッチテキストになり、受け取れない場所では`text/plain`へフォールバックします。

## HTMLは単純にする

Confluenceへ貼るHTMLには、複雑なCSSや独自属性を入れない方がよいです。

推奨される要素は次です。

```html
<h1>タイトル</h1>
<h2>概要</h2>

<p>
  この文章には<strong>重要な部分</strong>があります。
</p>

<ul>
  <li>項目A</li>
  <li>項目B</li>
</ul>

<table>
  <thead>
    <tr>
      <th>項目</th>
      <th>説明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>API</td>
      <td>使用不可</td>
    </tr>
  </tbody>
</table>
```

避けるべきものは次です。

```html
<div style="position:absolute">
<span class="custom-component">
<script>...</script>
<iframe>...</iframe>
```

Confluenceは貼り付け時に独自の文書モデルへ変換するので、細かなCSS、列幅、独自クラスなどは維持されない可能性があります。実際、表の列幅など、一部の表示情報がコピー＆ペーストで維持されない報告があります。([Atlassian Community](https://community.atlassian.com/forums/Confluence-questions/How-do-I-preserve-table-column-widths-in-copy-paste/qaq-p/1612729?utm_source=chatgpt.com))

---

# 5. AIにはConfluence用の中間記法を出させる

通常のMarkdownだけではConfluence固有要素を表せません。そこで、次のような目印を使います。

```markdown
# システム設計

## 注意事項

:::confluence-panel{type="warning"}
本番環境ではこの操作を実行しないでください。
:::

## 関連タスク

:::confluence-macro{type="jira"}
JQL: project = KAN AND statusCategory != Done
Columns: key, summary, status, assignee
:::

## 詳細

:::confluence-expand{title="内部実装"}
ここに詳細を書く。
:::
```

APIなしの場合、コンバーターはこれを完全なConfluenceマクロへ変換できません。代わりに、貼り付け結果を次のようにします。

```text
⚠ CONFLUENCE WARNING PANELに変換
本番環境ではこの操作を実行しないでください。

🔧 CONFLUENCE JIRA MACROを挿入
JQL: project = KAN AND statusCategory != Done
Columns: key, summary, status, assignee
```

人間は目印の場所で、

```text
/warning
/jira
/expand
/table of contents
/status
```

などを入力し、対応するConfluence要素へ置き換えます。

これにより、AIが特殊形式を直接扱わなくても、作業漏れを防げます。

---

# 6. 既存ページの編集方法

APIなしで既存ページを編集するときは、**ページ全体を上書きしない**ことが重要です。

## 推奨：見出し単位で置き換える

たとえば既存ページが、

```text
1. 概要
2. 背景
3. 設計
4. 運用
5. 参考資料
```

という構造なら、AIには「3. 設計」だけを渡します。

```text
次のConfluenceページの「3. 設計」セクションだけを更新してください。
他のセクションと整合する完成版を作り、
貼り付け可能なリッチテキストとして出力してください。

現在の内容:
...
```

その後、

1. Confluenceで編集モードを開く
2. 対象見出しの本文だけ選択
3. AIが作ったリッチテキストを貼り付け
4. 前後の見出しやマクロが壊れていないか確認
5. 更新メッセージを付けて公開

という手順にします。

## 追記の場合

追記はさらに安全です。

AIに、

```text
既存文章を書き換えず、末尾に追加する「検討結果」セクションを作ってください。
```

と指示し、新しい見出し以下を末尾へ貼ります。

## 削除・再構成の場合

ページ全体を大幅に変える場合は、元ページを直接変更せず、

```text
元ページ
└─ 「改訂案 2026-07-28」子ページ
```

を新規作成する方が安全です。

レビュー後に人間が差し替えるか、元ページをアーカイブします。

---

# 7. AI編集領域を最初から分ける

継続的にAIを使うなら、ページの構造を次のようにします。

```text
# プロジェクト設計書

## 人間が管理する基本方針
...

## AI更新対象：実装状況
...

## AI更新対象：未解決課題
...

## 人間が管理する承認記録
...
```

さらに見出し名を固定します。

```text
[AI-MANAGED] 実装状況
[AI-MANAGED] リスク一覧
[HUMAN-MANAGED] 意思決定
```

APIがなくても、どこを選択して貼り替えるべきか明確になります。

より安全なのは子ページ分割です。

```text
システム設計書
├─ 基本方針
├─ ADR
├─ AI生成：実装状況
├─ AI生成：コード調査結果
└─ AI生成：週次レポート
```

AI生成ページを独立させれば、既存の重要なページを壊す危険が減ります。

---

# 8. テンプレートを活用する

ページ作成権限があり、スペース管理者が協力してくれるなら、Confluenceテンプレートを用意するとかなり改善します。

テンプレート例：

```text
# タイトル

Status: DRAFT
Owner:
Last reviewed:

## 概要

[ここへAI生成文章を貼り付け]

## 背景

[ここへAI生成文章を貼り付け]

## 設計

[ここへAI生成文章を貼り付け]

## リスク

[ここへAI生成文章を貼り付け]

## 関連Jira

[Jiraマクロを設定]

## 変更履歴
```

人間が先に、

- 目次
- Jiraマクロ
- Page Properties
- Status
- レイアウト
- Infoパネル
- 固定リンク

を含むテンプレートを作ります。

AIはプレースホルダー部分の普通の文章だけを生成します。これならAIに特殊なConfluence形式を扱わせる必要がありません。

Confluenceではスペース管理者やサイト管理者がAutomationを作成でき、個人スペースでは本人がスペース管理者になっている場合もあります。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/create-automation-rules-in-confluence/?utm_source=chatgpt.com))

---

# 9. Confluence Automationが使える場合

APIトークンが禁止されていても、組織内のConfluence Automationは許可されている可能性があります。

Automationでは、

- 定期的なページ作成
- テンプレートからページ作成
- ラベル付与
- ページ移動
- 通知
- アーカイブ
- スマートボタン

などが可能です。Atlassianは、標準的なページツリーの作成や定型文書の自動生成をConfluence Automationの用途として案内しています。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/what-is-confluence-automation/?utm_source=chatgpt.com))

ただし、AutomationだけでAIが作った任意の長文を柔軟に挿入できるとは限りません。Jira Automationの「Create Confluence page」アクションについては、ページを作成できても本文を入力できないという公式上の制約があります。([Atlassian Support](https://support.atlassian.com/cloud-automation/docs/jira-automation-actions/?utm_source=chatgpt.com))

したがってAutomationは、

```text
空のページ・テンプレートを用意する
        ↓
人間がAI生成文章を貼り付ける
```

という補助に使うのが現実的です。

---

# 10. ブラウザ自動操作は可能だが、第二候補

Playwright、Selenium、Claudeのブラウザ操作、Chrome拡張機能などを使えば、APIなしでもブラウザ上で編集できます。

構成は次です。

```text
AIが文章を生成
  ↓
PlaywrightがConfluenceを開く
  ↓
編集ボタン
  ↓
対象セクションを選択
  ↓
リッチテキストを貼り付け
  ↓
プレビュー
  ↓
人間が公開ボタンを押す
```

ただし、完全自動化は推奨しません。

## 問題点

Confluenceのエディタは単純な`textarea`ではなく、構造化されたリッチテキストエディタです。そのため、DOMへ直接`innerHTML`を設定しても、エディタ内部の状態へ反映されないことがあります。

また、

- DOM構造変更で壊れる
- ツールバーやボタンのラベルが変わる
- 日本語・英語UIでセレクタが変わる
- ローディング状態に左右される
- 共同編集との競合
- 誤ったページを更新する危険
- 認証情報の管理
- 社内規則違反の可能性

があります。

## 実施するなら「半自動」

安全な境界は次です。

```text
自動化する
- ページを開く
- 編集モードにする
- 検索する
- 対象見出しまで移動する
- 文章を貼り付ける
- 差分を表示する

人間が行う
- 対象ページの最終確認
- マクロの確認
- 公開ボタン
```

本文入力には、DOMの直接書き換えではなく、実際のユーザー操作に近い方法を使います。

```text
1. エディタへフォーカス
2. クリップボードにtext/htmlを入れる
3. Cmd+V
4. エディタの表示を確認
```

この方がConfluence側の貼り付け変換処理を通るため、`innerHTML`操作より安全です。

---

# 11. Chrome拡張機能・ユーザースクリプトという中間案

社内で拡張機能の導入が認められるなら、専用の小さなChrome拡張機能が便利です。

機能は絞ります。

```text
Confluenceページで拡張ボタンを押す
    ↓
Markdown入力欄を開く
    ↓
ローカルでHTMLへ変換
    ↓
リッチテキストとしてクリップボードへコピー
    ↓
ユーザーが貼り付け
```

この方式なら、拡張機能がConfluenceへ直接書き込む必要はありません。

重要なのは、拡張機能の権限を最小限にすることです。

```json
{
  "permissions": ["clipboardWrite"],
  "host_permissions": []
}
```

可能なら、Confluenceページの読み取り権限さえ与えず、単なるMarkdown変換・コピー機能にします。

公開されている未知の拡張機能へ社内Confluenceのアクセス権限を与えるのは避けるべきです。外部拡張機能はページ内容や認証済みセッションへアクセスできる可能性があるため、社内審査済みのものか、自作の最小権限版が安全です。

---

# 12. 画像の扱い

画像はAPIなしだと別処理になります。

Atlassianの公式UIでは、

- 編集画面へドラッグ＆ドロップ
- ツールバーからアップロード
- `/files`
- `/images`

などで追加できます。画像をページへ挿入すると、ページの添付ファイルとして追加されます。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/display-files-and-images/?utm_source=chatgpt.com))

推奨フローは次です。

```text
AI文書内:
[IMAGE: architecture-overview.png]
Caption: システム全体構成

貼り付け後:
1. 目印の位置へ画像をドラッグ
2. キャプションを設定
3. 目印を削除
```

ローカル画像をHTMLの`<img src="file://...">`として貼っても、他の利用者が見られるConfluence添付にはなりません。

---

# 13. 表の扱い

単純な表はリッチテキスト貼り付けで比較的扱いやすいです。

```markdown
| 項目 | 状態 | 担当 |
|---|---|---|
| API | 未確認 | 山田 |
| Automation | 利用可能 | 佐藤 |
```

ただし、以下は崩れやすいです。

- セル結合
- 列幅
- セル背景色
- 複数段ヘッダー
- セル内の複雑なリスト
- 大量の列
- 画像入りセル

AIには次の制約を与えるとよいです。

```text
Confluenceへ貼り付ける表は、
最大5列、セル結合なし、セル内改行なし、
装飾なしの単純な表にしてください。
```

---

# 14. 推奨する実運用

ユーザーの状況では、次の構成が最も堅実です。

## フェーズ1：今すぐ使う

```text
1. AIに普通のMarkdownを書かせる
2. ChatGPTのレンダリング結果またはGoogle Docsを経由する
3. Confluenceへリッチテキスト貼り付け
4. 特殊要素だけ手作業で追加
5. 公開前に差分確認
```

## フェーズ2：作業を標準化する

社内用のプロンプトとテンプレートを作ります。

```text
- 見出し構成を固定
- 表を単純化
- マクロ挿入位置に目印
- AI管理セクションを固定
- 変更対象を見出し単位に限定
```

## フェーズ3：ローカル補助ツール

社内PC上だけで動く、

```text
Markdown → Confluence貼り付け用リッチテキスト
```

コンバーターを作ります。

必要機能はこの程度で十分です。

- Markdown入力
- プレビュー
- リッチテキストコピー
- マクロ目印
- 表の簡略化
- 危険なHTML除去
- 画像一覧の抽出
- 貼り付け後チェックリスト

## フェーズ4：許可が得られた場合のみ半自動化

- Playwrightでページを開く
- 対象セクションへ移動
- 貼り付けまで自動化
- 公開は人間

---

# 最適解の順位

1. **リッチテキストへ変換して手動貼り付け**
2. **新規長文ページはWord・Google Docsインポート**
3. **Confluenceテンプレート＋AI管理セクション**
4. **ローカル専用Markdown→Rich Textコンバーター**
5. **許可された場合のみ半自動ブラウザ操作**
6. **完全自動ブラウザ編集は原則避ける**

特におすすめなのは、

```text
AIはMarkdownだけを書く
Confluence専用コンバーターがリッチテキストを作る
人間が対象セクションへ貼り付けて公開する
```

という分担です。

これならAPI権限がなくても使え、ConfluenceのADFやStorage FormatをAIに生成させる必要もなく、誤更新や社内セキュリティ上のリスクも比較的小さくできます。

---

# you asked

message time: 2026-07-28 12:49:03

Conflience上で「ソースを表示」のようなボタンから完全なhtmlのような形式が見れていました。外で作ったこのような形式をインポートする機能はありますか？あるなら専用のコンバーターを作れます

---

# chatgpt response

その「ソースを表示」が標準機能なのか、Marketplaceアプリなのかで答えが変わります。Cloud／Data Center双方について、外部HTML・Storage Formatの取り込み可否と実装可能性を確認します。

はい、**その画面が編集可能な「Storage Format Source Editor」なら、外部で生成したConfluence Storage Formatを貼り付けて保存できます**。
この場合、APIなしでも専用コンバーターを作る価値はかなり高いです。

ただし、似た名前の機能が3種類あり、それぞれ挙動が違います。

# まず「ソース」の種類を判定する

表示された内容の冒頭や内部を確認してください。

## 1. Storage Format

次のようなXML風の内容なら、最も都合がよい形式です。

```xml
<h1>設計概要</h1>
<p>本文です。</p>

<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p>注意事項です。</p>
  </ac:rich-text-body>
</ac:structured-macro>

<ac:image>
  <ri:attachment ri:filename="architecture.png" />
</ac:image>
```

特徴は次です。

```text
<p>
<h1>
<table>
<ac:structured-macro>
<ac:parameter>
<ri:attachment>
<ri:page>
```

これはConfluenceの **Storage Format** です。厳密には普通のHTMLではなく、Confluence独自要素を含むXML/XHTML系の形式です。AtlassianもStorage Formatを「XHTML-based」と呼びつつ、独自のマクロ要素を含むため技術的にはXMLだと説明しています。([Atlassian Documentation](https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html))

## 2. ADF

次のようなJSONならAtlassian Document Formatです。

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [
        {
          "type": "text",
          "text": "本文です"
        }
      ]
    }
  ]
}
```

特徴は次です。

```text
"version": 1
"type": "doc"
"content": [...]
"type": "paragraph"
```

Marketplaceの一部Source Editorは、Storage FormatとADFの両方を編集できます。たとえばAppfireのSource Editorは、XHTMLまたはADF JSONを選び、貼り付け・検証・プレビュー・保存できると案内しています。([Atlassian Marketplace](https://marketplace.atlassian.com/apps/1215664/source-editor-for-confluence))

## 3. 「View Source」のエディタ表示形式

これが最も注意が必要です。

AtlassianはCloudについて、

> 「View Source」で見えるものは、ページのStorage Formatではなく、エディタパネルで使用される形式

と明記しています。([Atlassian Support](https://support.atlassian.com/atlassian-cloud/kb/how-to-retrieve-confluence-storage-format-on-cloud/))

つまり、メニュー名が単に「ソースを表示」「View Source」だった場合、それが必ずしも保存用Storage Formatとは限りません。

一方、Cloudで本当のStorage Formatを表示する標準手順は、

```text
ページ右上の …
→ Advanced details
→ View Storage Format
```

ですが、これはConfluence管理者権限が必要で、基本的には閲覧・コピー用途です。([Atlassian Support](https://support.atlassian.com/atlassian-cloud/kb/how-to-retrieve-confluence-storage-format-on-cloud/))

---

# 外部形式をインポートできるか

方法は大きく3つあります。

## 方法A：Source EditorへStorage Formatを直接貼る

**これが最も有力です。**

編集画面のツールバーに`<>`のようなSource Editorボタンがあり、開いた画面の内容を変更して保存できる場合、外部で生成したStorage Formatを貼り付けられます。

AtlassianのStorage Format Source Editorでは、ページのStorage Formatを直接編集し、マクロパラメータ、リンク、画像名、書式などを変更できます。Data Centerでは一部バージョンにバンドルされ、それ以外ではアプリとして導入されます。([Atlassian Documentation](https://confluence.atlassian.com/source/getting-started-with-storage-format-source-editor-277251157.html))

つまり、APIがなくても次のワークフローが成立します。

```text
Markdown
    ↓
専用コンバーター
    ↓
Confluence Storage Format
    ↓
クリップボードへコピー
    ↓
Source Editorへ貼り付け
    ↓
Preview
    ↓
保存
```

これは、以前説明したリッチテキスト貼り付けよりも正確です。

特に次の要素を生成できます。

- 見出し
- 表
- コードブロック
- パネル
- Expand
- Status
- Jiraマクロ
- 目次
- ページリンク
- 添付画像参照
- Marketplaceマクロ
- ページレイアウト

ただし、Source Editorが**編集可能であること**が前提です。単なる「表示」画面なら貼り戻せません。

---

## 方法B：ConfluenceのHTMLインポート機能

Confluence Cloudには、公式のHTMLインポート機能があります。

```text
Spaces
→ Import from other tools
→ HTML
→ HTMLを含むZIPをアップロード
```

ただし、これは期待する「現在のページにHTMLを流し込む機能」とは少し違います。

主な制約は次です。

- スペース作成権限が必要
- ZIPファイルとしてアップロードする
- HTMLファイル名がページ名になる
- 新しいスペースとして取り込む
- 既存ページの差分更新には向かない
- ConfluenceマクロをHTMLから復元できない
- 取り込める書式が限定される

公式インポーターは、見出し、段落、太字、リンク、画像、表、リスト、引用、インラインコードなどを保持します。一方、iframe、ボタン、数式、コードブロック、埋め込み動画などには制約があります。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/import-data-from-html-to-confluence/))

したがってHTMLインポートは、

```text
大量の既存HTMLページを一度だけConfluenceへ移行する
```

用途には使えますが、

```text
AIが作った文章を日常的に既存ページへ追加・編集する
```

用途にはあまり適しません。

---

## 方法C：リッチHTMLをクリップボード経由で貼る

これは前回説明した方式です。

```text
Markdown
→ HTML
→ text/htmlとしてクリップボードへコピー
→ 通常のConfluenceエディタへ貼り付け
```

Source Editorを編集できない場合は、この方法が現実的です。

ただし、Storage Formatの`ac:`や`ri:`要素を通常の編集画面へ貼り付けても、マクロとして認識されるとは限りません。

---

# 専用コンバーターは作れるか

**作れます。しかもSource Editorが使えるなら、かなり完成度の高いものを作れます。**

推奨構成は次です。

```text
┌───────────────────────────────┐
│ Markdown / AI生成テキスト      │
└──────────────┬────────────────┘
               ↓
┌───────────────────────────────┐
│ Markdown parser               │
│ remark / unified              │
└──────────────┬────────────────┘
               ↓
┌───────────────────────────────┐
│ 独自Canonical AST             │
│ Heading / Table / Panel / ... │
└──────────────┬────────────────┘
               ↓
┌───────────────────────────────┐
│ Confluence Storage Renderer   │
└──────────────┬────────────────┘
               ↓
┌───────────────────────────────┐
│ XML validation                │
│ Macro whitelist               │
└──────────────┬────────────────┘
               ↓
┌───────────────────────────────┐
│ Copy Storage Format           │
└───────────────────────────────┘
```

画面としては次のようなローカルWebアプリが便利です。

```text
┌────────────────────┬────────────────────┐
│ Markdown           │ Storage Format     │
│                    │                    │
│ # 設計概要         │ <h1>設計概要</h1> │
│                    │ <p>...</p>         │
│ :::info            │ <ac:structured...  │
│ 注意               │                    │
├────────────────────┴────────────────────┤
│ Preview | Validate | Copy Storage       │
└─────────────────────────────────────────┘
```

# Markdown側の独自記法

通常のMarkdownだけではマクロを表現できないので、拡張記法を設けます。

## パネル

```markdown
:::panel{type="info" title="補足"}
ここに説明を書く。
:::
```

変換結果：

```xml
<ac:structured-macro ac:name="info">
  <ac:parameter ac:name="title">補足</ac:parameter>
  <ac:rich-text-body>
    <p>ここに説明を書く。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

## Expand

```markdown
:::expand{title="詳細を見る"}
詳細な説明。
:::
```

変換結果：

```xml
<ac:structured-macro ac:name="expand">
  <ac:parameter ac:name="title">詳細を見る</ac:parameter>
  <ac:rich-text-body>
    <p>詳細な説明。</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

## Jira一覧

```markdown
:::jira
jql: project = KAN AND statusCategory != Done
columns: key,summary,status,assignee
maximumIssues: 20
:::
```

変換側では、JQL文字列や最大件数を検証してから、正しいマクロXMLを生成します。

## Status

```markdown
:status[レビュー中]{color="Yellow"}
```

## ページリンク

```markdown
[設計書](confluence-page:システム設計書)
```

変換結果：

```xml
<ac:link>
  <ri:page ri:content-title="システム設計書" />
  <ac:link-body>設計書</ac:link-body>
</ac:link>
```

---

# 最初に行うべき小さな実験

専用ツールを作る前に、次のテストを行えば利用可能性を判断できます。

## テスト1：ソースが編集可能か

1. テスト用ページを作る
2. Source Editorを開く
3. `<p>テスト文章</p>`を末尾に追加
4. Previewまたは保存
5. ページへ反映されるか確認

反映されれば、外部コンバーター方式が使えます。

## テスト2：マクロを外部生成できるか

通常エディタでInfoパネルを1個作り、Source Editorを開きます。

表示された該当部分をコピーし、もう一つ複製します。

```xml
<ac:structured-macro ac:name="info">
  ...
</ac:structured-macro>
```

保存後にInfoパネルが2個になれば、マクロも生成できます。

## テスト3：全文置換できるか

元ソースをローカルにバックアップしてから、単純なページで次を貼ります。

```xml
<h1>変換テスト</h1>
<p>これは外部で作成したStorage Formatです。</p>
<ul>
  <li><p>項目1</p></li>
  <li><p>項目2</p></li>
</ul>
```

これが保存できれば、Markdown→Storageコンバーターを実用化できます。

---

# 重要な注意点

## 普通のHTMLをそのまま貼るわけではない

次のHTMLはStorage Formatでも概ね使えます。

```html
<h2>見出し</h2>
<p><strong>太字</strong>です。</p>
```

しかし、次のようなWebページ用HTMLは使えません。

```html
<!DOCTYPE html>
<html>
<head>
  <style>...</style>
  <script>...</script>
</head>
<body>
...
</body>
</html>
```

Storage Formatには通常、次を含めません。

- `<!DOCTYPE>`
- `<html>`
- `<head>`
- `<body>`
- `<script>`
- 任意のJavaScript
- 任意のCSS
- iframe
- Reactコンポーネント

基本的には**ページ本文の断片**だけです。

```xml
<h1>タイトル</h1>
<p>本文</p>
```

## XMLとして正しくする

HTMLでは許される曖昧な記述も、Storage Formatでは失敗することがあります。

```html
<!-- 避ける -->
<br>
<img src="x">

<!-- XMLとして閉じる -->
<br />
<img src="x" />
```

特殊文字もエスケープします。

```text
&  → &amp;
<  → &lt;
>  → &gt;
"  → &quot;  属性内
```

## 添付画像は別問題

次のStorage Formatを生成しても、

```xml
<ac:image>
  <ri:attachment ri:filename="diagram.png" />
</ac:image>
```

ページに`diagram.png`という添付ファイルが存在しなければ画像は表示されません。

APIなしの場合は、

1. 通常エディタで画像をページにアップロード
2. ファイル名を固定
3. Storage Formatからそのファイル名を参照

という二段階になります。

## Marketplaceマクロ

Marketplaceマクロは、マクロ名、パラメータ、schema version、macro IDなどが複雑な場合があります。

最も安全なのは、

1. 通常エディタで一度マクロを作る
2. Source EditorでStorage Formatを取得
3. そのXMLをテンプレートとして保存
4. 許可した値だけ置換する

方法です。

AIにゼロから推測させるべきではありません。

---

# 既存ページを編集する場合

全文をAIに再生成させて置き換えるのは危険です。

専用コンバーターには、次の機能を持たせるべきです。

## セクションの抽出

```text
既存Storage Format
→ 見出し単位で分割
→ 編集対象だけMarkdownへ変換
```

## セクションの差し替え

```text
AIが更新したMarkdown
→ Storage Format
→ 対象見出しの範囲だけ置換
→ その他は元XMLを保持
```

たとえば、

```json
{
  "operation": "replace_section",
  "heading": "実装方針",
  "contentMarkdown": "..."
}
```

という編集指示をAIに返させます。

外部コンバーターは、元のStorage Formatから「実装方針」の開始位置と次の同レベル見出しを探し、その範囲だけ置換します。

これなら、

- 既存マクロ
- 添付画像
- ページリンク
- 人間の編集領域
- 未知のMarketplace要素

を保持できます。

---

# 最終判断

ユーザーが見たものが、

### 編集可能なStorage Format Source Editor

なら、最適な構成はこれです。

```text
AI
→ 拡張Markdown
→ 独自コンバーター
→ Storage Format
→ Source Editorへ貼り付け
→ Preview
→ 人間が保存
```

APIが使えない環境としては、かなり強力な方法です。

### 読み取り専用のView Storage Format

なら、Storage Formatを取得することはできますが、直接インポートはできません。その場合は、

```text
Markdown
→ リッチHTML
→ 通常エディタへ貼り付け
```

を使います。

### 通常の「View Source」

なら、それがStorage Formatではない可能性があります。`ac:structured-macro`や`ri:attachment`が含まれるかを確認する必要があります。

そして公式HTMLインポートも存在しますが、これは**新しいスペースへの一括移行用**であり、日々のページ作成・編集用コンバーターの出力先としては適していません。HTMLインポートにはスペース作成権限が必要で、対応書式も限定されています。([Atlassian Support](https://support.atlassian.com/confluence-cloud/docs/import-data-from-html-to-confluence/))

したがって、**Source Editorに書き戻せる環境なら、Markdown→Confluence Storage Formatコンバーターを作る方針が最も有望**です。

---
Powered by [AI Exporter](https://saveai.net)