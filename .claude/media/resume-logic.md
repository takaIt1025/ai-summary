# 再開ロジック

`automation/logs/progress.json` を読み込み、以下のルールで開始Phaseを決定する。

```
phase 値              → 開始するPhase
─────────────────────────────────────
""（空）または "analyzed" → Phase 1 から新規開始
"kw_done"               → Phase 2 から再開
"research_done"          → Phase 3 から再開
"design_done"            → Phase 4 から再開
"write_done"             → Phase 5 から再開
"quality_done"           → Phase 6 から再開
"published"              → Phase 7 から再開
```

## 再開時の確認手順

1. 再開Phaseの【インプット】ファイルが存在するか確認する
2. 存在しない場合は1つ前のPhaseから再実行する
3. `errors` フィールドにエラーが残っている場合はユーザーに内容を報告してから再開する
