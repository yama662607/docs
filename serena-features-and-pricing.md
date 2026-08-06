> From: https://chatgpt.com/c/6a677fe1-d170-83ee-9cb9-73fa6ed70a05

# you asked

message time: 2026-07-28 00:57:21

Serenaの機能について調べてjetbrain pluginはいくら？intelliJが使える企業なら使える？

---

# chatgpt response

Serena本体の機能、JetBrains連携の形、料金・ライセンス条件を公式情報中心に確認します。特に「企業のIntelliJライセンスがあれば追加費用なしで使えるか」を分けて整理します。

## 結論

**Serena本体は無料・オープンソース**です。  
ただし、JetBrainsのコード解析エンジンをSerenaから利用するための **Serena JetBrains Pluginは別売り**で、現在のJetBrains Marketplace表示では **1ユーザーあたり年額US$70**です。無料トライアルもあります。([JetBrains](https://www.jetbrains.com/store/plugins/?utm_source=chatgpt.com))

したがって、

> 会社がIntelliJ IDEAを契約しているだけでは、Serena JetBrains Pluginは自動的には使えません。

会社側でSerenaプラグインのライセンスを追加購入するか、個人ライセンスを業務利用してよいか社内規定を確認する必要があります。

---

# Serenaとは何か

Serenaは、Claude Code、Codex、Gemini CLI、Cursor、JetBrains AI Assistant、JunieなどのAIエージェントに、**IDEのようなコード理解・編集能力を追加するMCPサーバー**です。

通常のAIエージェントは、次のような探索をしがちです。

```text
grepで文字列検索
↓
ファイル全体を読む
↓
関係ありそうな場所を推測
↓
文字列置換で修正
```

Serenaを使うと、コードを関数、クラス、メソッド、型などの**シンボル単位**で扱えます。

```text
UserServiceクラスを検索
↓
そのクラスを参照しているシンボルを取得
↓
対象メソッドの本体だけ読む
↓
シンボル単位で置換・挿入
```

これにより、大きなリポジトリでも、AIがファイルを片端から読む必要が減り、コンテキスト消費と誤編集を抑えられます。([GitHub](https://github.com/oraios/serena))

## 主な機能

### コード探索

- シンボル検索
- ファイル内のクラス・関数一覧取得
- 参照元・使用箇所の検索
- 宣言・実装の検索
- IDEの診断やInspection情報の取得
- プロジェクト構造の把握

### コード編集

- 関数やクラスの本体を置換
- シンボルの前後へコードを挿入
- シンボル名の変更
- 安全な削除

### エージェント向け機能

- プロジェクトごとのオンボーディング
- リポジトリ情報を保存するMemory
- グローバル、プロジェクト、クライアント単位の設定
- Claude Code、Codex、Gemini CLIなどとのMCP接続

Serena自体はLLMではないので、実際に判断して作業するClaude CodeやCodexなどは別途必要です。([GitHub](https://github.com/oraios/serena))

---

# 無料版とJetBrains Plugin版の違い

Serenaには大きく2種類のコード解析バックエンドがあります。

| 項目 | 無料LSP版 | JetBrains Plugin版 |
|---|---:|---:|
| Serena本体 | 無料 | 無料 |
| 解析バックエンド | Language Server | JetBrains IDE |
| 料金 | 基本無料 | 年額US$70/ユーザー |
| シンボル検索 | 可能 | 可能 |
| 参照検索 | 可能 | より安定 |
| 外部ライブラリ検索 | 制限あり | 対応 |
| 型階層 | 基本なし | 対応 |
| シンボル移動 | 基本なし | 対応 |
| Inline refactoring | 基本なし | 対応 |
| ファイル・ディレクトリRename | 制限あり | 対応 |
| 対話的デバッグ | なし | 対応 |
| 複数言語混在プロジェクト | 言語サーバー依存 | JetBrainsの解析能力を利用 |
| Language Server設定 | 必要な場合あり | 原則不要 |

無料版でも40種類以上の言語に対応しており、通常のシンボル検索・参照検索・シンボル編集は利用できます。JetBrains版では、依存ライブラリの索引、型階層、移動・インライン化などの高度なリファクタリング、デバッガー操作が加わります。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

## JetBrains Plugin固有の重要機能

### 1. 外部ライブラリまで検索できる

無料LSP版では、プロジェクト内のコードは追えても、ライブラリ内部の宣言や型情報を完全には追えない場合があります。

JetBrains版では、IDEがインデックスしている依存ライブラリもSerenaから参照できます。

たとえばJavaなら、

```text
Spring Framework内部の型
Gradle/Maven依存ライブラリ
継承元インターフェース
フレームワークが認識するシンボル
```

までエージェントが検索できます。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

### 2. 高度なリファクタリング

JetBrains版ではIDEのリファクタリング機構を利用して、

- クラス・関数・ファイルの移動
- シンボルのインライン化
- Renameに伴う参照箇所の更新
- 未使用コードを伝播的に削除
- 型階層の取得

などが可能です。単純な検索置換より安全性が高くなります。([GitHub](https://github.com/oraios/serena))

### 3. エージェントがデバッガーを操作できる

JetBrains Plugin版では、AIエージェントが次の操作を行えます。

- ブレークポイント設定
- プログラム実行・停止・ステップ実行
- 変数の確認
- 式の評価
- 実行フローの制御

永続的なREPL形式のインターフェースを通して操作する仕組みで、JetBrains Plugin専用機能です。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

---

# 料金

2026年7月時点でJetBrains Marketplaceのストア表示は次のとおりです。

> **Serena: US$70 / user / year**

無料トライアルがあります。([JetBrains Marketplace](https://plugins.jetbrains.com/vendor/oraios-ai?utm_source=chatgpt.com))

日本円での実際の請求額は、購入時の為替、税、JetBrains側の地域別価格、会社契約の見積もり条件によって変わります。

なお、過去には個人向け月額US$5という案内・販売形態も確認できますが、現在の公開ストアで明確に確認できる標準価格は年額US$70です。年間契約にすると、単純換算では月あたり約US$5.83です。([GitHub](https://github.com/oraios/serena/issues/952?utm_source=chatgpt.com))

---

# 会社がIntelliJを使える場合、Serenaも使えるのか

## IntelliJのライセンスとは別

JetBrains Marketplaceの有料プラグインは、IntelliJ IDEA本体とは別の商品です。

会社が次のどれを持っていても、

- IntelliJ IDEA Ultimate
- All Products Pack
- JetBrains IDEの法人ライセンス
- JetBrains AI Assistant
- Junie

Serena Pluginの権利が自動的に含まれるとは限りません。

Serena Pluginは、Oraios Softwareが販売する**サードパーティー製有料プラグイン**で、JetBrains Marketplace経由の別契約です。EULA上も「Paid Plugin」として、購入したPlugin User数の範囲内で利用する必要があります。([GitHub](https://github.com/oraios/serena-jetbrains-plugin-public/blob/main/LICENSE.md))

## 企業で使うために必要な条件

基本的には次の3点です。

1. **対応するJetBrains IDEを使えること**
2. **Serena Pluginの利用者分のライセンスがあること**
3. **会社のセキュリティ・OSS・AI利用規定で許可されていること**

たとえば開発者10人が同時にSerena Pluginを利用するなら、原則として10人分のPlugin Userライセンスが必要です。EULAでは、購入したPlugin Users数を超えないようにする義務が定められています。([GitHub](https://github.com/oraios/serena-jetbrains-plugin-public/blob/main/LICENSE.md))

## 会社が購入する場合

JetBrains Marketplaceには法人向けの「Get quote」が用意されているため、会社管理のJetBrainsアカウントやライセンス管理システムから人数分を購入・割り当てる形が自然です。([JetBrains](https://www.jetbrains.com/store/plugins/?utm_source=chatgpt.com))

## 個人購入したものを会社で使う場合

技術的にはJetBrains Accountに紐づくライセンスとして認証できる可能性がありますが、次の確認が必要です。

- 個人ライセンスの業務利用条件
- 会社PCへの個人JetBrains Accountログイン可否
- 会社の経費精算規定
- 有料IDEプラグインの持ち込み可否
- ソースコードをAIエージェントへ渡すことの可否

ライセンス上だけでなく、会社の情報セキュリティ規定が重要です。

---

# IntelliJが無料版でも使えるか

Serena公式ドキュメントでは、JetBrains IDEを普段のエディタとして使っていなくても、**無料のCommunity Editionを裏で起動し、コード解析バックエンドとして利用できる**と案内されています。

つまり、

```text
普段の編集：VS Code / Cursor / Zed
解析バックエンド：IntelliJ IDEA Community / PyCharm Community
AIエージェント：Claude Code / Codex
MCP：Serena
```

という構成も可能です。JetBrains IDEで対象プロジェクトを開き、インデックス済みにしておく必要があります。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

ただし、**IDE本体が無料でもSerena Pluginの料金は別途必要**です。

---

# 対応IDE

公式には次のJetBrains系IDEが例示されています。

- IntelliJ IDEA
- PyCharm
- Android Studio
- WebStorm
- PhpStorm
- RubyMine
- GoLand

JetBrains IDEが認識できる言語・フレームワークを基本的に利用できます。

一方、現時点では次は非対応です。

- Rider
- CLion

([Oraios Software](https://oraios-software.de/serena_jetbrains_plugin.php))

---

# 実際の構成

Serena JetBrains Pluginは、JetBrains IDE内で直接AIチャットを提供するプラグインではありません。

構造は次のようになります。

```text
Claude Code / Codex / Junie / JetBrains AI Assistant
                  │
                  │ MCP
                  ▼
          Serena MCP Server
                  │
                  │ ローカル接続
                  ▼
       Serena JetBrains Plugin
                  │
                  ▼
      IntelliJのインデックス・解析・デバッガー
```

普段の指示はClaude CodeやCodexなどに出します。Serena Pluginは、その裏側でJetBrains IDEのコード解析機能をSerenaへ提供します。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

設定は概ね次のようになります。

```bash
serena init -b JetBrains
```

または、

```yaml
# ~/.serena/serena_config.yml
language_backend: JetBrains
```

JetBrains IDEとSerenaで開くプロジェクトルートは一致させる必要があります。([OraiOS](https://oraios.github.io/serena/02-usage/025_jetbrains_plugin.html))

---

# あなたの用途に対する判断

あなたが考えているような、

- 大規模リポジトリ全体の探索
- PRレビュー用のコンテキスト収集
- Claude CodeやCodexへの正確なシンボル情報提供
- ライブラリ・依存関係まで含めた解析
- 複数エージェントによる並列探索
- 安全なRename、Move、Inline
- デバッガーを使った原因調査

では、JetBrains Plugin版の価値はかなり高いです。

特に無料LSP版との差が大きいのは、**依存ライブラリの索引、高度なリファクタリング、対話的デバッグ、複数言語プロジェクトの安定性**です。

一方、最初から会社に購入を依頼しなくても、まず無料のSerena本体＋LSPバックエンドで、

- シンボル検索の精度
- Claude Code/Codexとの相性
- トークン削減効果
- リポジトリ探索速度

を評価できます。その後、JetBrains版の無料トライアルで差を比較するのが合理的です。

**最終的な整理は次のとおりです。**

| 質問 | 回答 |
|---|---|
| Serena本体はいくら？ | 無料、オープンソース |
| JetBrains Pluginはいくら？ | 現在の表示でUS$70/ユーザー/年 |
| 無料トライアルはある？ | ある |
| IntelliJ契約に含まれる？ | 原則含まれない |
| 会社のIntelliJがあれば動く？ | 技術的には動くが、Serena Pluginの別ライセンスが必要 |
| IntelliJ Communityでも使える？ | 公式上は利用可能 |
| Claude CodeやCodexと使える？ | MCP経由で利用可能 |
| Rider、CLionは？ | 現時点では非対応 |

---
Powered by [AI Exporter](https://saveai.net)