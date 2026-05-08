# Phase 3: 設計 ⚠️ 人間確認ポイント

```
【インプット】
- progress.json: { "phase": "research_done", "kw": "確定KW" }
- data/x_trends/x_enriched_article{N}.json
- data/youtube_transcripts/ 以下のテキストファイル

【アウトプット】
- articles/draft_{article_id}_structure.md（確定H2構成）
- progress.json 更新: { "phase": "design_done" }

【完了条件】
- articles/draft_{article_id}_structure.md が存在すること
- AUTO_MODE=false の場合: ユーザー承認済みであること
```

## 実行手順

1. **検索上位10記事のH2/H3構造抽出**
   - Ahrefs `serp-overview` で上位10記事のURL / DR / タイトルを取得
   - 全10サイト（最低7サイト）のページをフェッチしてH2/H3見出しを抽出

2. **共通パターン分析**
   - **共通60-70%**: 出現率50%以上のH2は踏襲（検索意図の核心）
   - **独自30-40%**: 出現率25%以下 or 0%のH2を一次情報から設計（差別化ポイント）

3. **一次情報の配置確定**
   - 確定H2ごとに「使うX投稿」「使うYouTube動画」を具体的に決定

4. **⚠️ ユーザー確認（AUTO_MODEに従う）**
   ```
   AUTO_MODE=false: 確定H2構成を提示 → ユーザーの「承認」入力を待つ
   AUTO_MODE=true:  そのまま次へ進む
   ```

5. **状態更新**
   - `articles/draft_{article_id}_structure.md` に確定構成を保存
   - `progress.json` を更新
