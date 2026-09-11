# Codex Desktop マルチプロバイダ構成ガイド (OpenCodex)

Codex Desktop の UI をそのまま使い、Kimi / DeepSeek / GLM (Coding Plan) / Gemini (Google 契約枠) を
モデルピッカーから選べるようにする手順。2026-09 に実機で検証済み (macOS, Codex Desktop 26.901, codex 0.153.3, OpenCodex 2.42.0)。

## 仕組み (なぜ動くか)

- Codex Desktop は同梱 codex バイナリを `codex app-server` (JSON-RPC) で駆動し、設定は CLI と共通の `~/.codex/config.toml`
- `config.toml` の `openai_base_url` をローカルプロキシ (OpenCodex, port 10100) に向けると、Codex には Responses API のまま、上流で各社形式 (Chat Completions / Anthropic Messages / Gemini native / cloud-code OAuth) に変換される
- `model_catalog_json` を注入すると Desktop のモデルピッカーに候補が現れる (非カタログモデルは `Custom` 表示)

## 1. インストール

```bash
# 前提: bun (mise 管理), git
git clone --depth 1 https://github.com/lidge-jun/opencodex.git ~/Code/Projects/opencodex
cd ~/Code/Projects/opencodex && bun install
bun link                      # 登録側
cd ~ && bun link @bitkyc08/opencodex   # ~/.bun/bin/ocx ができる
ocx --version                 # 動作確認
```

更新は clone 内で `git pull` するだけ (bun link はパス参照)。開発が非常に活発でほぼ日次リリースのため、破壊的変更に注意。

## 2. API キー登録 (値を画面に出さない方法)

キーは `~/.config/mise/config.local.toml` に置く (この Mac の規約)。値を表示せず stdin パイプで登録:

```bash
eval "$(sed -n 's/^\([A-Za-z_][A-Za-z_0-9]*\)[[:space:]]*=[[:space:]]*["\x27]\?\([^"\x27]*\)["\x27]\?/\1='\''\2'\''/p' ~/.config/mise/config.local.toml)"

printf '%s\n' "$DEEPSEEK_API_KEY" | ocx login deepseek      # DeepSeek (従量)
printf '%s\n' "$KIMI_API_KEY"     | ocx login moonshot      # Kimi (従量)
printf '%s\n' "$GLM_API_KEY"      | ocx login zai           # GLM Coding Plan (契約枠)
```

各社のダッシュボードがブラウザで開くが無視してよい。`valid ✅` が出れば保存済み。
キーの実体は `~/.opencodex/config.json`、OAuth トークンは `~/.opencodex/auth.json`。

## 3. 起動と注入

```bash
ocx start     # プロキシ起動 + Codex 設定注入 (~/.codex/config.toml に marker ブロックで 2 行追加のみ)
ocx sync      # プロバイダからモデル検出して ~/.codex/opencodex-catalog.json を再生成
```

バックアップ: `ocx` は自前で復元を持つ (`ocx uninstall` で完全復元)。
念のため `cp -a ~/.codex/config.toml ~/.codex/auth.json <backup-dir>/` も推奨。

## 4. Gemini を subscription (Google 契約枠) で使う

`GEMINI_API_KEY` は AI Studio 従量課金。Pro/Ultra 契約枠を使うには OAuth:

```bash
ocx login google-antigravity   # ブラウザで Google ログイン (再ログインも同コマンド)
```

API キー版と OAuth 版の両方がカタログに現れるので、従量課金側を非表示にする (→ §5)。

## 5. モデルの絞り込み (ピッカー整理)

```bash
ocx models list                          # 現在のカタログ確認
ocx models disable <provider/model>      # 非表示 (visibility: hide。削除ではない)
ocx models enable  <provider/model>      # 再表示
ocx provider edit <name> --default-model <id>   # プロバイダ既定モデル
ocx sync                                 # 変更後は再注入
```

検証時の最終構成 (8 モデル):

```
OpenAI:   gpt-6-astra / gpt-5.6-sol / gpt-5.6-terra / gpt-5.6-luna  (ネイティブパススルー)
DeepSeek: v4-pro
Gemini:   google-antigravity/gemini-3.8-flash   (OAuth = 契約枠。API キー版 google/* は disable)
Kimi:     k3
GLM:      glm-5.3-flash
```

## 6. 常駐化

```bash
ocx service install     # launchd 登録。Mac 再起動後も自動起動
ocx service status      # 状態 / ログ: ~/.opencodex/service.log
ocx service restart / stop
```

セットアップ後は Codex Desktop を**完全再起動**し、新規セッションでモデルピッカーを確認。

## 運用メモ

- **Codex Desktop 更新でモデルが消える**: Desktop 更新が `config.toml` を再生成して注入が消えるため。
  `ocx sync` で復旧 (新 codex バイナリも `openai_base_url` / `model_catalog_json` を継続サポートすることを確認済み)
- **ピッカーのモデル名は slug** (`zai/glm-5.3-flash` 等)。`ocx alias` で短縮名を張れる
- **thinking effort**: カタログに effort 一覧が入る (例: kimi-k3 は max/ultra 固定)。
  `~/.codex/config.toml` の `model_reasoning_effort` よりピッカー指定が優先
- **使用状況**: `ocx usage --range today` でプロバイダ別リクエスト/トークン/概算コスト

## 既知の制約

1. **collab (サブエージェント) の暗号化**: 親セッションがネイティブ ChatGPT モデル (astra 等) のとき、
   `spawn_agent` 等の引数は ChatGPT バックエンドが暗号化するため、サブエージェントにサードパーティモデルを
   指定できない。混合協調は「routed モデルを親にしたセッション」で行う
   (詳細: codex-desktop-enhance/docs/codex-collab-tools.md)
2. **Claude subscription (Pro/Max) の流用は非推奨**: Anthropic は自社クライアント外の OAuth を
   サーバ側でブロックしており、プロキシは Claude Code フィンガープリント偽装で回避する実装。
   OpenCodex 自身が「最高位の ToS リスク」と明記。API キー (従量) か正規クライアント (Claude Code) を使う
3. **タスク途中のモデル切替**: セッション単位。切り替えは新規セッションで行う
4. Desktop UI は署名済みのため改造不可。介入は config.toml + カタログ経由のみに留めるのが堅牢
