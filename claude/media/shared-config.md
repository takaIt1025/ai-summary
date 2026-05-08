# 共通設定（エラーハンドリング・状態管理）

## 共通エラーハンドリング（全Phaseに適用）

| エラー種別 | 対応 |
|---|---|
| API 503 / 429 | 10分間隔で最大3回リトライ → 失敗時はerrorに記録してPhaseをスキップ |
| API 401 / 403 | 即停止。ユーザーに「[Phase名] 認証エラー: [API名]」を報告して終了 |
| スプレッドシートAPI障害 | progress.jsonにローカル記録して続行。復旧後に同期 |
| タイムアウト（30秒超） | 1回リトライ → 失敗時はスキップして次ステップへ |
| 不明なエラー | `automation/logs/error.log` に記録してユーザーに報告 |

## progress.json スキーマ

```json
{
  "article_id": "042",
  "phase": "design_done",
  "kw": "Claude Code 使い方",
  "title": "",
  "slug": "",
  "published_url": "",
  "git_commit": null,
  "deploy_status": null,
  "quality_score": null,
  "auto_mode": false,
  "completed_phases": ["kw_done", "research_done", "design_done"],
  "errors": [],
  "last_updated": "2026-05-09T10:00:00"
}
```

## phaseの値

```
kw_done → research_done → design_done → write_done → quality_done → published → analyzed
```

## 強制リセット

```bash
echo '{"article_id":"","phase":"","kw":"","title":"","slug":"","published_url":"","git_commit":null,"deploy_status":null,"quality_score":null,"auto_mode":false,"completed_phases":[],"errors":[],"last_updated":""}' > automation/logs/progress.json
```
