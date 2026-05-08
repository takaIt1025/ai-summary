# Phase 6: 公開

```
【インプット】
- progress.json: { "phase": "quality_done", "quality_score": 95以上 }
- articles/final_{article_id}.md

【アウトプット】
- WordPressに記事が公開済み
- progress.json 更新: { "phase": "published", "wp_post_id": 1234, "published_url": "https://..." }

【完了条件】
- wp_post_id と published_url が progress.json に記録されていること
- AUTO_MODE=false の場合: ユーザーが下書き確認後に公開指示を出したこと
```

## 実行手順

1. **アイキャッチ画像生成**
   - フォールバック順: NanoBanana Pro → NanoBanana Flash → Gemini Flash
   - 16:9 / フラットイラスト / カテゴリカラー準拠

2. **H2直下の図解画像生成**（3-5枚）
   - プロンプトに必ず付与: `No English text whatsoever. Japanese text only or no text at all.`
   - 生成後に英語テキスト混入がないか目視確認。混入があれば再生成

3. **Markdown→HTML変換**
   ```bash
   python3 -c "
   import markdown
   with open('articles/final_ARTICLE_ID.md') as f:
       md = f.read()
   html = markdown.markdown(md, extensions=['tables','extra'])
   print(html)
   " > automation/images/article_ARTICLE_ID.html
   ```
   - Gutenbergブロックは変換前に退避し、変換後に復元

4. **WordPress投稿（下書き）**
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/posts" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"title":"タイトル","content":"HTML本文","status":"draft","categories":[ID],"slug":"slug","excerpt":"メタ"}'
   # レスポンスの "id" を wp_post_id として progress.json に記録
   ```

5. **画像アップロード**
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/media" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Disposition: attachment; filename=eyecatch.png" \
     --data-binary @automation/images/eyecatch_ARTICLE_ID.png
   # レスポンスの "id" を featured_media に設定
   ```

6. **Rank Math SEO設定**
   ```bash
   curl -X POST "${WP_URL}/wp-json/rankmath/v1/updateMeta" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"objectID":WP_POST_ID,"objectType":"post","meta":{"rank_math_title":"SEOタイトル","rank_math_description":"メタディスクリプション","rank_math_focus_keyword":"フォーカスKW","rank_math_robots":["index","follow"]}}'
   ```

7. **FAQ Schema設定**
   ```bash
   curl -X POST "${WP_URL}/wp-json/rankmath/v1/updateSchemas" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"objectID":WP_POST_ID,"objectType":"post","schemas":{"schema-FAQPage-1":{"schema-type":"FAQPage","schema-name":"FAQPage-1","schema-data":{"@type":"FAQPage","mainEntity":[{"@type":"Question","name":"Q1","acceptedAnswer":{"@type":"Answer","text":"A1"}}]}}}}'
   ```

8. **公開（⚠️ AUTO_MODEに従う）**
   ```
   AUTO_MODE=false: 下書きURLをユーザーに提示 → 「公開」入力を待つ
   AUTO_MODE=true:  品質スコア95点以上を確認して以下を実行
   ```
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/posts/${WP_POST_ID}" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"status":"publish"}'
   ```

9. **状態更新**
   - `progress.json` に `wp_post_id` と `published_url` を記録して更新
