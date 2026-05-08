# SEOオウンドメディア 全自動パイプライン

## 実行モード（$ARGUMENTSで指定）
```
/media-pipeline AUTO_MODE=false  # デフォルト。Phase 3でユーザー確認あり
/media-pipeline AUTO_MODE=true   # 全自動。確認なしで公開まで実行
```

## 共通設定ファイル
- 状態管理: `automation/logs/progress.json`
- エラー/再開ロジック: Read `.claude/media/shared-config.md`
- 品質ルール: Read `.claude/media/quality-rules.md`（Phase 4/5で参照）

## 実行手順

### 開始時（毎回必ず）
1. `.claude/media/shared-config.md` を Read してエラーハンドリング・progress.json スキーマを把握する
2. `.claude/media/resume-logic.md` を Read して開始Phaseを決定する
3. AUTO_MODE を確定する（$ARGUMENTS → progress.json の auto_mode → デフォルト false の優先順）

### Phase実行
開始Phaseが決まったら、そのPhaseのファイルを Read してから実行する。
**各Phaseファイルは実行直前に読む。全部を先読みしない。**

| Phase | ファイル | 概要 |
|---|---|---|
| 1 | `.claude/media/phase1-kw.md` | KW選定（Ahrefs・3C分析） |
| 2 | `.claude/media/phase2-research.md` | リサーチ（X API・YouTube） |
| 3 | `.claude/media/phase3-design.md` | 設計（上位10記事分析・確認ポイント） |
| 4 | `.claude/media/phase4-write.md` | 執筆（ドラフト・AI感排除） |
| 5 | `.claude/media/phase5-quality.md` | 品質チェック（5視点95点） |
| 6 | `.claude/media/phase6-publish.md` | 公開（画像生成・Astro git push・デプロイ待機） |
| 7 | `.claude/media/phase7-analyze.md` | 分析（インデックス・KPI） |

### 各Phase共通ルール
- Phase開始時: `automation/logs/progress.json` を読み込む
- Phase完了時: `progress.json` を更新してから次に進む
- エラー発生時: `shared-config.md` のエラーハンドリングに従う

## 自動スケジュール
| スケジュール | 実行時刻(JST) | 内容 |
|---|---|---|
| Morning Pipeline | 毎日 5:00 | Phase 1〜7 / AUTO_MODE=true |
| Afternoon Pipeline | 毎日 14:00 | Phase 1〜7 / AUTO_MODE=true |
| Daily KPI Report | 毎日 22:13 | Phase 7のKPIレポートのみ |
| Weekly Optimize | 月曜 10:23 | `.claude/media/weekly-optimize.md` を参照 |
