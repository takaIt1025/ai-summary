# Phase 6: 公開（Astro）

```
【インプット】
- progress.json: { "phase": "quality_done", "quality_score": 95以上 }
- articles/final_{article_id}.md

【アウトプット】
- {ASTRO_PROJECT_PATH}/src/content/blog/{slug}.md（記事ファイル）
- {ASTRO_PROJECT_PATH}/public/images/{article_id}/（画像ファイル群）
- git push → CI/CDが自動デプロイ
- progress.json 更新: { "phase": "published", "git_commit": "abc123", "slug": "slug", "published_url": "https://..." }

【完了条件】
- git push が成功していること
- AUTO_MODE=false の場合: ユーザーが内容確認後に push 指示を出したこと
- deploy_status が "deployed" になっていること（URL疎通確認済み）
```

> **前提環境変数（.envに追加）**
> ```bash
> ASTRO_PROJECT_PATH=/path/to/your-astro-project
> SITE_URL=https://YOUR_DOMAIN.com
> DEPLOY_BRANCH=main
> ```

## 実行手順

1. **frontmatterの生成**

   articles/final_{article_id}.md の内容をもとにAstroのfrontmatterを生成する。

   ```yaml
   ---
   title: "記事タイトル（30文字以内・KWを先頭に）"
   description: "メタディスクリプション（120文字以内）"
   pubDate: 2026-05-09
   updatedDate: 2026-05-09
   slug: "url-slug"
   category: "カテゴリ名"
   tags: ["タグ1", "タグ2"]
   eyecatch: "/images/ARTICLE_ID/eyecatch.png"
   faq:
     - question: "Q1"
       answer: "A1"
     - question: "Q2"
       answer: "A2"
   draft: false
   ---
   ```

2. **記事ファイルの書き込み**

   ```bash
   # frontmatter + 本文を結合してAstroプロジェクトに書き込む
   cat > "${ASTRO_PROJECT_PATH}/src/content/blog/${SLUG}.md" << 'ARTICLE'
   [frontmatter + articles/final_{article_id}.md の本文]
   ARTICLE
   ```

3. **画像の生成・配置**

   - アイキャッチ: NanoBanana Flash または Gemini Flash（16:9・フラットイラスト）
     - フォールバック: NanoBanana Pro → NanoBanana Flash → Gemini Flash
   - H2直下の図解（3-5枚）
     - プロンプトに必ず付与: `No English text whatsoever. Japanese text only or no text at all.`
     - 生成後に英語テキスト混入がないか目視確認。混入があれば再生成

   ```bash
   mkdir -p "${ASTRO_PROJECT_PATH}/public/images/${ARTICLE_ID}"
   cp automation/images/eyecatch_${ARTICLE_ID}.png \
      "${ASTRO_PROJECT_PATH}/public/images/${ARTICLE_ID}/eyecatch.png"
   cp automation/images/fig_${ARTICLE_ID}_*.png \
      "${ASTRO_PROJECT_PATH}/public/images/${ARTICLE_ID}/"
   ```

4. **⚠️ ユーザー確認（AUTO_MODEに従う）**

   ```
   AUTO_MODE=false:
     生成した記事ファイルのパスをユーザーに提示 → 「push」入力を待つ

   AUTO_MODE=true:
     品質スコア95点以上を確認してそのまま次へ
   ```

5. **git push（デプロイトリガー）**

   ```bash
   cd "${ASTRO_PROJECT_PATH}"
   git add "src/content/blog/${SLUG}.md" "public/images/${ARTICLE_ID}/"
   git commit -m "feat: ${TITLE}"
   git push origin ${DEPLOY_BRANCH}
   GIT_COMMIT=$(git rev-parse HEAD)
   ```

6. **デプロイ完了確認（最大5分待機）**

   ```bash
   PUBLISHED_URL="${SITE_URL}/blog/${SLUG}/"
   MAX_WAIT=300
   ELAPSED=0
   until curl -s -o /dev/null -w "%{http_code}" "${PUBLISHED_URL}" | grep -q "200"; do
     sleep 30
     ELAPSED=$((ELAPSED + 30))
     echo "デプロイ待機中... ${ELAPSED}秒経過"
     if [ $ELAPSED -ge $MAX_WAIT ]; then
       echo "タイムアウト: 手動でデプロイ状況を確認してください"
       break
     fi
   done
   ```

   - タイムアウト時: `deploy_status: "timeout"` として記録してPhase 7へ進む
   - Phase 7内でIndexing APIをリトライする

7. **状態更新**

   ```json
   {
     "phase": "published",
     "slug": "url-slug",
     "published_url": "https://YOUR_DOMAIN.com/blog/url-slug/",
     "git_commit": "abc123...",
     "deploy_status": "deployed"
   }
   ```
