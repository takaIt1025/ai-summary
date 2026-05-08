# Phase 2: リサーチ

```
【インプット】
- progress.json: { "phase": "kw_done", "kw": "確定KW" }

【アウトプット】
- data/x_trends/x_enriched_article{N}.json
- data/youtube_transcripts/{videoId}.txt
- progress.json 更新: { "phase": "research_done" }

【完了条件チェック】（次に進む前に必ず実行）
```bash
ls data/x_trends/x_enriched_article*.json 2>/dev/null | wc -l  # 30以上
ls data/youtube_transcripts/*.txt 2>/dev/null | wc -l           # 5以上
```
```

## 実行手順

1. **X一次情報の収集**
   - X API v2で記事KWに関連する投稿を検索（直近1ヶ月・min_impressions: 500）
   - 必ず取得するフィールド: `note_tweet`（長文全文）/ スレッド全文 / メディアURL / 投稿URL
   - 30件以上を収集して `data/x_trends/x_enriched_article{N}.json` に保存
   - スプレッドシート「X一次情報」タブに記録（G列は200字以上の全文）

2. **YouTube一次情報の収集**
   - YouTube Data API v3で関連動画を検索（上位10本）
   - `yt-dlp` で自動字幕を取得:
     ```bash
     yt-dlp --write-auto-sub --sub-lang ja --skip-download \
       -o "data/youtube_transcripts/%(id)s" "VIDEO_URL"
     ```
   - スプレッドシート「YouTube一次情報」タブに記録
   - **H列（文字起こし完了）が全件「○」になるまで次に進まない**

3. **完了確認・状態更新**
   - 上記【完了条件チェック】コマンドで件数を確認する
   - 基準未満の場合はAPIを再試行する（shared-config.mdのエラーハンドリングに従う）
   - `progress.json` を更新
