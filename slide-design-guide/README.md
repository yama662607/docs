# 汎用スライド・ポスター設計ガイド（AIスライド作成スキル向け知識ベース）

特定のツールや生成スクリプトに依存せず、**Codex / Claude Code / 汎用AIエージェント**でスライドや学術ポスターを作成する際にそのまま参照・組み込みできる、デザイン・構成・言語表現・品質レビューの体系的リファレンスです。

認知科学・知覚心理学・データ可視化研究・学術発表規範・コンサルティング実務・AI生成デザイン批評（Anti-Slop）の実証知見と一次出典をベースに整理しています。

## ドキュメント一覧

| ファイル | 内容 | 主な使いどころ |
|---|---|---|
| [01-universal-principles.md](./01-universal-principles.md) | **普遍的デザイン原則** | 認知負荷理論（CLT）・Mayer 12原理・Kosslyn 8原則・Tufte・Gestalt・WCAG・Assertion-Evidence の基礎と15の普遍的禁止事項 |
| [02-genre-profiles-and-visual-system.md](./02-genre-profiles-and-visual-system.md) | **ジャンル別プロファイル＆視覚システム** | 口頭発表 vs 読む資料の切り分け、品質計画5要素、配色・カバー・タイトル帯・レイアウト構成、学術/ビジネス/講義別の作法 |
| [03-content-planning-and-storyboard.md](./03-content-planning-and-storyboard.md) | **構成設計・ストーリーボード・根拠管理** | 主張＋ブリッジ設計、ゴーストデック/HTMLモック先行、図の契約（Figure Contract）、数値の出所分類、既存テンプレ編集時の鉄則 |
| [04-layout-geometry-and-anti-slop.md](./04-layout-geometry-and-anti-slop.md) | **レイアウト幾何学＆AI病（Anti-Slop）排除** | 余白・ギャップ・2カラム制限・左右役割分担・カードパディング・表設計規約・典型的な「AI臭レイアウト」禁止集 |
| [05-japanese-typography-and-ai-smell-lexicon.md](./05-japanese-typography-and-ai-smell-lexicon.md) | **日本語組版＆AI臭ワード・言い回し排除集** | 和欧フォント設計・Mac/Win互換の注意点・改行と表記揺れ・日本語のAI臭語彙/構文/トーン・30秒セルフチェック |
| [06-visual-exhibits-and-data-viz.md](./06-visual-exhibits-and-data-viz.md) | **図表・グラフ・数式・引用の設計規範** | 5大作図手法の選び方、論文図の再描画、誤差棒（SD/SE/95%CI）・知覚均等カラーマップ（viridis/Okabe-Ito）、数式と他者図版クレジット |
| [07-qa-review-and-checklist.md](./07-qa-review-and-checklist.md) | **品質検査・独立レビュー・最終テスト** | 数値基準表、機械的欠陥/デザイン/アクセシビリティ検査、独立レビュー手順と採否表、納品前4テスト＆汎用レビュープロンプト |
| [08-academic-poster-guide.md](./08-academic-poster-guide.md) | **学術ポスター（A0/A1/A2）設計ガイド** | 1〜2m視認前提の3カラムグリッド・タイプスケール・300dpi画像・5部構成・印刷入稿/CMYK/QR/A4モックチェック |
| [references-and-research-sources.md](./references-and-research-sources.md) | **参考文献・一次資料リンク集** | 本ガイドの根拠となった論文・大学ガイドライン・デザイン批評・実務記事のURL一覧 |

## 他のスライド作成スキルへ組み込む際の推奨ポイント

自分のスライド作成スキル（`SKILL.md` / `CLAUDE.md` / `AGENTS.md` / `references/`）へ取り込む場合は、特に以下の5点をルール化すると効果が大きくなります。

1. **「見せる資料」と「読ませる資料」を最初に確定する**（02, 03）
2. **デザイン前に「タイトル（主張）＋根拠＋前後ブリッジ」の骨子で通し読み検証を行う**（03）
3. **AI臭レイアウト（紫グラデ、均等カード並べ、片側カラーバー、3連続同一レイアウト）とAI臭ワードを禁止する**（04, 05）
4. **投影可読性（本文18pt以上・下線禁止・コントラスト4.5:1以上・色だけに頼らない）を固定ルールにする**（01, 05, 07）
5. **生成後に文脈を伏せた別視点（Fresh-Eye Review）と各種チェックリストで破綻を検査する**（07）
