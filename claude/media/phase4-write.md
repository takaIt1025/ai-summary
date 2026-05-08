# Phase 4: 執筆

```
【インプット】
- progress.json: { "phase": "design_done" }
- articles/draft_{article_id}_structure.md
- data/x_trends/ および data/youtube_transcripts/ の一次情報

【アウトプット】
- articles/draft_{article_id}.md
- progress.json 更新: { "phase": "write_done" }

【完了条件】
- articles/draft_{article_id}.md が存在すること
- 文字数 5,000字以上
- AI感排除7項目チェックが全て ✅（quality-rules.md 参照）
```

> 品質ルールの詳細は `.claude/media/quality-rules.md` を参照すること。

## 実行手順

1. **80%ドラフト生成**
   - 確定H2構成に沿って本文を生成
   - 一次情報を自然に組み込む（全体の20%以上）
   - 独自ファクト3箇所以上 / 目標文字数 5,000字以上

2. **AI感排除チェック（7項目）**
   - quality-rules.md の「AI感排除7項目」に従い全項目チェック・修正

3. **文章ルール適用**
   - quality-rules.md の「文章ルール」に従い適用

4. **X引用スタイル**
   - 自分の体験・感想を先に述べ、後から「〜氏も同様のことを言っています」と添える
   - blockquoteブロック使用禁止。インラインで自然に組み込む
   - @username は `<a href="https://x.com/username" target="_blank" rel="noopener">@username</a>` 形式

5. **SEO Knowledge品質批判10-20周**
   - 検索意図との整合性 / E-E-A-T / 内部リンク候補 / 独自性 / 読者が行動できるか

6. **状態更新**
   - `articles/draft_{article_id}.md` に保存
   - `progress.json` を更新
