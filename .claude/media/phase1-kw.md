# Phase 1: KW選定

```
【インプット】
- automation/logs/progress.json
- スプレッドシート「記事作成ログ」の未着手行（障害時はスキップ可）

【アウトプット】
- 確定KW 1つ
- progress.json 更新: { "phase": "kw_done", "kw": "確定KW", "article_id": "NNN" }

【完了条件】
- progress.json の phase が "kw_done" かつ kw が空でないこと
```

## 実行手順

1. **記事作成ログの確認**
   - スプレッドシート「記事作成ログ」の「未着手」行の一番上を取得
   - 未着手行がある → そのKWで Phase 2 に進む（Ahrefsリサーチ不要）
   - 未着手行がない → 以下のKW選定を実行

2. **競合KW抽出（3C分析）**
   - Ahrefs `site-explorer-organic-competitors` で競合を自動発見
   - 競合3-5サイトの `site-explorer-organic-keywords` でKW抽出
   - フィルター: country=JP, volume>=200, position<=20

3. **KWデータ取得**
   - Ahrefs `keywords-explorer-overview` で Volume / KD / CPC / Intent を取得

4. **SEO Knowledge批判3周**（以下9観点で3周実施）
   1. ロングテール優先（Vol 1,000以下）に沿っているか
   2. CV距離（Do/Buy優先）の分類は正しいか
   3. カニバリゼーションリスクはないか（parent_topic確認）
   4. KD値の信頼性（SERP上位のDRで検証）
   5. トピカルオーソリティ設計（クラスター先→ピラー後）
   6. 競合SERPでDR低サイトが勝てている実績はあるか
   7. 独自性・E-E-A-Tを出せるKWか
   8. 攻め順序は妥当か
   9. 抜けているKWカテゴリはないか

5. **状態更新**
   - `progress.json` を更新
   - スプレッドシート「KW戦略」「記事作成ログ」を更新（障害時はスキップして後で同期）
