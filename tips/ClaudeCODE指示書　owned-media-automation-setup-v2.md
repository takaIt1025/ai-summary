# SEOオウンドメディア 全自動パイプライン構築ガイド v2

> **このファイルについて**
> 元ファイル（v1）の構造的問題を改善したバージョン。元ファイルは `owned-media-automation-setup.md` に保存済み。
> 主な改善点: ①AUTO_MODEの明示 ②共通エラーハンドリング ③各PhaseのINPUT/OUTPUT定義 ④状態管理のローカル化 ⑤品質チェックの役割分離

---

## ⚙️ 実行モード設定（必ず最初に確認）

```
AUTO_MODE = false  （デフォルト）
  → Phase 3でユーザー確認待ち。確認後に次Phaseへ進む。

AUTO_MODE = true
  → Phase 3の確認をスキップ。品質95点以上で自動公開まで実行。
```

**実行時の指定方法:**
```
# 手動確認モード（推奨・初期運用）
/media-pipeline AUTO_MODE=false

# 全自動モード（安定稼働後）
/media-pipeline AUTO_MODE=true
```

---

## 🚨 共通エラーハンドリング（全Phaseに適用）

各Phaseで以下のエラーが発生した場合、個別のエラー処理より本セクションを優先する。

| エラー種別 | 対応 |
|---|---|
| API 503 / 429 | 10分間隔で最大3回リトライ → 3回失敗時はエラーログに記録してPhaseをスキップ |
| API 401 / 403 | 即停止。ユーザーに「[Phase名] 認証エラー: [API名]」を報告して終了 |
| スプレッドシートAPI障害 | `automation/logs/progress.json` にローカル記録して続行。復旧後に同期 |
| タイムアウト（30秒超） | 1回リトライ → 失敗時はスキップして次ステップへ |
| 不明なエラー | エラー内容を `automation/logs/error.log` に記録してユーザーに報告 |

---

## 📋 状態管理（progress.json）

各PhaseはこのファイルをINPUT/OUTPUTの橋渡しとして使用する。
スプレッドシートAPIが落ちていても、このファイルが最低限の状態を保証する。

```json
{
  "article_id": "042",
  "phase": "phase3_done",
  "kw": "Claude Code 使い方",
  "title": "",
  "wp_post_id": null,
  "quality_score": null,
  "auto_mode": false,
  "completed_phases": ["phase1", "phase2", "phase3"],
  "errors": [],
  "last_updated": "2026-05-08T10:00:00"
}
```

**phaseの値:**
```
kw_done → research_done → design_done → write_done → quality_done → published → analyzed
```

---

## 1. 概要

### このシステムが行うこと
SEOオウンドメディアの記事制作を、KW選定から公開後処理まで全自動で実行する。

### 自動化される工程
1. **Phase 1 / KW選定**: Ahrefs APIで競合KW抽出→3C分析→SEO Knowledge批判3周→確定
2. **Phase 2 / リサーチ**: X API v2で関連投稿30件+YouTube文字起こし5-10本
3. **Phase 3 / 設計**: 検索上位10記事のH2構造分析→共通60-70%+独自30-40%の構成確定
4. **Phase 4 / 執筆**: 80%ドラフト→一次情報20%組み込み→AI感排除→SEO Knowledge品質批判10-20周
5. **Phase 5 / 品質**: 5視点採点（95点以上が公開条件）
6. **Phase 6 / 公開**: アイキャッチ+H2直下図解→WordPress自動投稿→公開
7. **Phase 7 / 分析**: インデックス登録→スプレッドシート更新→内部リンク自動追加→KPIレポート

### 期待される成果
- 毎日2本、月60本ペースで高品質SEO記事を公開
- 人間は品質チェックの最終承認と方向性決定のみ

---

## 2. 前提条件

### 必須ツール/API

| ツール | 用途 | 備考 |
|:------|:----|:----|
| Claude Code（Anthropic CLI） | パイプライン実行基盤 | Scheduled Tasks または cron で定期実行 |
| WordPress サイト | 記事公開先 | REST API有効化必須。Rank Math推奨 |
| Ahrefs API（MCP経由） | KW調査・競合分析・検索上位分析 | MCP設定が必要 |
| X(Twitter) API v2 | 一次情報収集（投稿） | Bearer Token必須。note_tweetフィールド対応 |
| YouTube Data API v3 | 一次情報収集（動画） | APIキー必須 |
| Google Sheets API | データ管理・KPIレポート | MCP経由推奨 |
| Google Indexing API | 公開後の即時インデックス登録 | サービスアカウント必須 |
| AI画像生成ツール（NanoBanana MCP or Gemini API） | 画像生成 | NanoBanana推奨 |
| yt-dlp | YouTube自動字幕取得 | `pip install yt-dlp` |
| Python 3.10+ | Markdown→HTML変換 | `markdown` ライブラリ必要 |

### 認証情報（.envファイルに設定）

```bash
WP_URL=https://YOUR_DOMAIN.com
WP_USER=YOUR_WP_USERNAME
WP_PASSWORD=YOUR_WP_APPLICATION_PASSWORD
X_BEARER_TOKEN=YOUR_X_BEARER_TOKEN
X_API_KEY=YOUR_X_API_KEY
X_API_SECRET=YOUR_X_API_SECRET
YOUTUBE_API_KEY=YOUR_YOUTUBE_API_KEY
GOOGLE_SHEETS_CREDENTIALS_PATH=./credentials.json
GOOGLE_SHEETS_TOKEN_PATH=./sheets-token.json
INDEXING_SERVICE_ACCOUNT_PATH=./indexing-service-account.json
AHREFS_API_KEY=YOUR_AHREFS_API_KEY
NANOBANANA_API_KEY=YOUR_NANOBANANA_KEY
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
GA4_PROPERTY_ID=YOUR_GA4_PROPERTY_ID
GSC_SITE_URL=YOUR_GSC_SITE_URL
SPREADSHEET_ID=YOUR_SPREADSHEET_ID
```

---

## 3. セットアップ手順

### 3.1 ディレクトリ構成の作成

```
your-media-project/
├── CLAUDE.md
├── .env
├── articles/
├── automation/
│   ├── images/
│   ├── logs/
│   │   ├── progress.json   ← 状態管理（新規追加）
│   │   └── error.log       ← エラーログ（新規追加）
│   └── design/
├── data/
│   ├── x_trends/
│   └── youtube_transcripts/
├── kpi_feedback.md
└── wp_additional_css.css
```

```bash
mkdir -p articles automation/{images,logs,design} data/{x_trends,youtube_transcripts}
echo '{"article_id":"","phase":"","kw":"","title":"","wp_post_id":null,"quality_score":null,"auto_mode":false,"completed_phases":[],"errors":[],"last_updated":""}' > automation/logs/progress.json
touch .env kpi_feedback.md automation/logs/error.log
```

### 3.2 以降の設定
v1と同一。`owned-media-automation-setup.md` の Section 3.2〜3.5 を参照。

---

## 4. 自動化パイプライン定義（全7Phase）

> **共通ルール**
> - 各Phaseの開始時に `automation/logs/progress.json` を読み込む
> - 各Phaseの完了時に `progress.json` を更新してから次に進む
> - エラー発生時は冒頭「共通エラーハンドリング」セクションに従う

---

### Phase 1: KW選定

```
【インプット】
- automation/logs/progress.json
- スプレッドシート「記事作成ログ」タブの未着手行（任意・障害時はスキップ）

【アウトプット】
- 確定KW 1つ
- progress.json 更新: { "phase": "kw_done", "kw": "確定KW", "article_id": "NNN" }
- スプレッドシート「KW戦略」「記事作成ログ」更新

【完了条件】
- progress.json の phase が "kw_done" になっていること
- kw フィールドが空でないこと
```

**目的**: 次に書く記事のキーワードを決定する。

**実行手順**:

1. **記事作成ログの確認**
   - スプレッドシート「記事作成ログ」タブのステータスが「未着手」の一番上の行を取得
   - 未着手行がある場合 → そのKWで Phase 2 に進む（Ahrefsリサーチ不要）
   - 未着手行がない場合 → 以下の手順でKW選定を実行

2. **競合KW抽出（3C分析）**
   - Ahrefs `site-explorer-organic-competitors` で自サイトの競合を自動発見
   - 競合3-5サイトの `site-explorer-organic-keywords` でKW抽出
   - フィルター: country=JP, volume>=200, position<=20
   - 自社の強み（Company）で独自価値を出せるKWを優先

3. **KWデータ取得**
   - Ahrefs `keywords-explorer-overview` で Volume / KD / CPC / Intent を取得

4. **SEO Knowledge批判3周**
   以下の9観点で批判的レビューを3周実施:
   1. ロングテール優先（Vol 1,000以下）に沿っているか
   2. CV距離（Do/Buy優先）の分類は正しいか
   3. カニバリゼーションリスクはないか（parent_topic確認）
   4. KD値の信頼性（SERP上位のDRで検証）
   5. トピカルオーソリティ設計（クラスター先→ピラー後）
   6. 競合SERPでDR低サイトが勝てている実績はあるか
   7. 独自性・E-E-A-Tを出せるKWか
   8. 攻め順序は妥当か
   9. 抜けているKWカテゴリはないか

5. **KW確定・状態更新**
   - `progress.json` を更新
   - スプレッドシート更新（障害時はスキップ、後で同期）

---

### Phase 2: リサーチ

```
【インプット】
- progress.json: { "phase": "kw_done", "kw": "確定KW" }

【アウトプット】
- data/x_trends/x_enriched_article{N}.json（X投稿データ）
- data/youtube_transcripts/{videoId}.txt（文字起こし）
- progress.json 更新: { "phase": "research_done" }
- スプレッドシート「X一次情報」「YouTube一次情報」更新

【完了条件】
- X投稿: 30件以上収集済み
- YouTube文字起こし: 5本以上完了
- progress.json の phase が "research_done" になっていること
```

**目的**: 記事に独自性を持たせるための一次情報を収集する。

**実行手順**:

1. **X一次情報の収集**
   - X API v2で記事KWに関連する投稿を検索（直近1ヶ月・min_impressions: 500）
   - 必ず取得するフィールド: `note_tweet`（長文全文）、スレッド全文、メディアURL、投稿URL
   - 30件以上を目標に収集
   - `data/x_trends/x_enriched_article{N}.json` に保存

2. **YouTube一次情報の収集**
   - YouTube Data API v3で関連動画を検索（上位10本）
   - `yt-dlp` で自動字幕を取得:
     ```bash
     yt-dlp --write-auto-sub --sub-lang ja --skip-download -o "data/youtube_transcripts/%(id)s" "VIDEO_URL"
     ```

3. **状態更新**
   - `progress.json` を更新

---

### Phase 3: 設計（⚠️ 人間確認ポイント）

```
【インプット】
- progress.json: { "phase": "research_done", "kw": "確定KW" }
- data/x_trends/x_enriched_article{N}.json
- data/youtube_transcripts/ 以下のテキストファイル

【アウトプット】
- articles/draft_{article_id}_structure.md（確定H2構成）
- progress.json 更新: { "phase": "design_done" }

【完了条件】
- H2構成ファイルが存在すること
- AUTO_MODE=false の場合: ユーザー承認済みであること
- progress.json の phase が "design_done" になっていること
```

**目的**: 検索上位記事の構造を分析し、共通パターン+独自パートで記事構成を確定する。

**実行手順**:

1. **検索上位10記事のH2/H3構造抽出**
   - Ahrefs `serp-overview` で検索上位10記事のURL / DR / タイトルを取得
   - 全10サイト（最低7サイト）のページをフェッチし、H2/H3見出しを抽出

2. **共通パターン分析**
   - **共通60-70%**: 出現率50%以上のH2は踏襲
   - **独自30-40%**: 出現率25%以下 or 0%のH2を一次情報から設計

3. **一次情報の配置確定**
   - 確定H2ごとに使うX投稿・YouTube動画を具体的に決定

4. **⚠️ ユーザー確認（AUTO_MODE設定に従う）**
   ```
   AUTO_MODE=false: H2構成を提示 → ユーザーの「承認」入力を待つ → Phase 4へ
   AUTO_MODE=true:  H2構成を progress.json に記録して自動的に Phase 4へ
   ```

5. **状態更新**
   - `articles/draft_{article_id}_structure.md` に確定構成を保存
   - `progress.json` を更新

---

### Phase 4: 執筆

```
【インプット】
- progress.json: { "phase": "design_done" }
- articles/draft_{article_id}_structure.md（確定H2構成）
- data/x_trends/ および data/youtube_transcripts/ の一次情報

【アウトプット】
- articles/draft_{article_id}.md（記事本文）
- progress.json 更新: { "phase": "write_done" }

【完了条件】
- 記事本文ファイルが存在すること
- 文字数 5,000字以上
- AI感排除7項目チェックが全て ✅ であること（下記「品質ルール」参照）
- progress.json の phase が "write_done" になっていること
```

**目的**: 確定した構成に基づいて高品質な記事本文を生成する。

**実行手順**:

1. **80%ドラフト生成**
   - 確定H2構成に沿って本文を生成
   - 各H2に紐付けた一次情報を自然に組み込む（全体の20%以上）
   - 独自ファクト3箇所以上を含める
   - 目標文字数: 5,000字以上

2. **AI感排除チェック（7項目）**
   → 詳細は本ファイル末尾「品質ルール」セクションを参照
   - [ ] あいまいな結論がない
   - [ ] 「〜することが重要です」が3回以下
   - [ ] 文末パターンに変化がある
   - [ ] 3連続箇条書きがない
   - [ ] 「これにより」が削除済み
   - [ ] 「おすすめします」→「してください」に置換済み
   - [ ] 一人称の体験談が2箇所以上

3. **文章ルール適用**
   → 詳細は本ファイル末尾「品質ルール」セクションを参照

4. **X引用スタイル適用**
   - 自分の体験・感想を先に述べる
   - blockquoteブロック使用禁止。インラインで自然に組み込む
   - @username は `<a href="https://x.com/username" target="_blank" rel="noopener">@username</a>` 形式

5. **SEO Knowledge品質批判10-20周**
   - 検索意図との整合性 / E-E-A-T / 内部リンク候補 / 独自性 / 読者が行動できるか

6. **状態更新**
   - `articles/draft_{article_id}.md` に保存
   - `progress.json` を更新

---

### Phase 5: 品質チェック

```
【インプット】
- progress.json: { "phase": "write_done" }
- articles/draft_{article_id}.md（記事本文）

【アウトプット】
- articles/final_{article_id}.md（修正済み最終稿）
- progress.json 更新: { "phase": "quality_done", "quality_score": 97 }

【完了条件】
- 5視点合計スコアが 95点以上
- progress.json の phase が "quality_done" になっていること
```

> ⚠️ **役割切り替え（重要）**
> このPhaseに入る前に、以下を必ず実行すること:
>
> 「あなたはPhase 4で記事を書いたエージェントとは**完全に別の存在**です。
> 今から**厳格な編集長**として振る舞い、この記事を初めて読む視点で採点してください。
> 執筆プロセスへの配慮は一切不要。読者の利益のみを基準に採点すること。」

**目的**: SEO最適化と多角的品質チェックで95点以上を達成する。

**実行手順**:

1. **メタ情報設定**
   - タイトル: 30文字以内、KWを先頭に配置、数字を含める
   - メタディスクリプション: 120文字以内

2. **E-E-A-T注入**
   - 著者情報・データ引用・事例を追加

3. **内部リンク・外部リンク**
   - 内部リンク: 3-5本 / 外部権威ソース: 3-5本

4. **GEO/LLMO構造対応**
   - 各H2冒頭に1文結論 / FAQ Schema 5問以上

5. **5視点採点（各20点 / 合計100点）**

   **視点1: デザイン（フォーマット）【20点】**
   - H2構成・テーブルスタイル・CTA配置・図解・strong配置・段落・箇条書き・コードブロック・altテキスト・統一性（各2点）

   **視点2: SEO【20点】**
   - タイトルKW・メタ文字数・5000字以上・H2冒頭結論・外部リンク権威・E-E-A-T・FAQ数・内部リンク数・検索意図整合・独自ファクト数（各2点）

   **視点3: 編集（文体）【20点】**
   - AI感排除・ですます統一・リード文・体験談・X引用・専門用語説明・文長・文末バリエーション・体言止め・「わかること」ボックス（各2点）

   **視点4: 技術正確性【20点】**
   - ツール説明・プロンプト実用性・手順粒度・専門概念説明・限界記載・事例数値・比較正確性・次のアクション・最新動向・ユースケース具体性（各2点）

   **視点5: 読者UX（ペルソナ: 中小企業経営者・ITリテラシー中程度）【20点】**
   - リード共感・専門用語説明・H2結論・再現性・次のアクション明確・段落読みやすさ・表の完結性・FAQ整合・CTA文脈・読後感（各2点）

6. **修正ループ**
   ```
   採点 → 合計 < 95点:
     1. 最低スコアの視点の指摘を優先修正
     2. 修正した視点のみ再採点
     3. 最大3サイクル繰り返す
     4. 3サイクル後も95点未満 → ユーザーに差分レポートを提出して停止
   ```

7. **状態更新**
   - `articles/final_{article_id}.md` に最終稿を保存
   - `progress.json` を更新

---

### Phase 6: 公開

```
【インプット】
- progress.json: { "phase": "quality_done", "quality_score": 95以上 }
- articles/final_{article_id}.md（最終稿）

【アウトプット】
- WordPressに記事が公開済み
- progress.json 更新: { "phase": "published", "wp_post_id": 1234 }

【完了条件】
- WordPress REST APIでステータスが "publish" になっていること
- wp_post_id が progress.json に記録されていること
- AUTO_MODE=false の場合: ユーザーが下書き確認後に公開指示を出したこと
```

**目的**: アイキャッチ画像と記事内図解を生成し、WordPressに公開する。

**実行手順**:

1. **アイキャッチ画像生成**
   - NanoBanana Flash または Gemini Flash で生成（16:9 / フラットイラスト）
   - フォールバック順: NanoBanana Pro → NanoBanana Flash → Gemini Flash

2. **H2直下の図解画像生成**
   - プロンプトに必ず付与: 「No English text whatsoever. Japanese text only or no text at all.」
   - 生成後に英語テキスト混入がないか目視確認。混入があれば再生成

3. **Markdown→HTML変換**
   - Python `markdown` ライブラリで変換
   - Gutenbergブロックは変換前に退避し、変換後に復元

4. **WordPress投稿（下書き）**
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/posts" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"title":"タイトル","content":"HTML本文","status":"draft","categories":[ID],"slug":"slug","excerpt":"メタ"}'
   ```

5. **画像アップロード・Rank Math SEO設定・FAQ Schema設定**
   → v1の手順と同一（Section 4 Phase 6 の手順5〜7を参照）

6. **公開（⚠️ AUTO_MODEに従う）**
   ```
   AUTO_MODE=false: 下書きURLをユーザーに提示 → 「公開」入力を待つ
   AUTO_MODE=true:  品質スコア95点以上を確認して自動公開
   ```

7. **状態更新**
   - `progress.json` に wp_post_id を記録して更新

---

### Phase 7: 分析

```
【インプット】
- progress.json: { "phase": "published", "wp_post_id": 1234 }

【アウトプット】
- Googleインデックス登録完了
- スプレッドシート全タブ更新
- kpi_feedback.md 更新
- progress.json 更新: { "phase": "analyzed" }

【完了条件】
- Indexing APIへのリクエストが成功していること（またはGSC手動登録依頼済み）
- progress.json の phase が "analyzed" になっていること
```

**目的**: インデックス登録、スプレッドシート更新、内部リンク追加、KPIレポート生成。

**実行手順**:
→ v1の手順と同一（Section 4 Phase 7 を参照）。
ただし各ステップ完了後に `progress.json` の `errors` フィールドにエラーを記録すること。

---

## 5. 自動スケジュール定義

| スケジュール | 実行時刻(JST) | 内容 |
|:----------|:------------|:----|
| Morning Pipeline | 毎日 5:00 | Phase 1〜7 / AUTO_MODE=true |
| Afternoon Pipeline | 毎日 14:00 | Phase 1〜7 / AUTO_MODE=true |
| Daily KPI Report | 毎日 22:13 | Phase 7のKPIレポートのみ |
| Weekly Optimize | 月曜 10:23 | カニバリ検出・リライト・内部リンク最適化 |

### パイプライン実行フロー
```
1. kpi_feedback.md を読み込む（前日の成功/失敗パターンを反映）
2. progress.json を確認（前回中断していた場合は途中から再開）
3. Phase 1〜7 を順番に実行
4. 各Phase完了後に progress.json を更新
5. 全Phase完了後に完了通知
```

---

## 6. 品質ルール（単一参照元）

> ⚠️ Phase 4 と Phase 5 の両方がこのセクションを参照する。
> 同じ内容を2箇所に書かない。ルール変更はここだけ修正すること。

### AI感排除7項目（全記事で必ずチェック）

| # | チェック項目 | NG例 | OK例 |
|:--|:-----------|:----|:----|
| 1 | あいまいな結論 | 「場合もあります」 | 具体的シナリオ・条件を明示 |
| 2 | 重要ですの連発 | 「重要です」が4回以上 | 記事中3回まで |
| 3 | 文末パターン同一 | 全段落「〜です。」 | 「〜です」「〜ます」「体言止め」を交互に |
| 4 | 3連続箇条書き | 箇条書き3連続 | 間にリード文かテーブルを挟む |
| 5 | これにより | 「これにより効率が向上します」 | 完全削除。具体的な因果関係を記述 |
| 6 | おすすめします | 「検討することをおすすめします」 | 「してください」に置換 |
| 7 | 一人称の欠如 | 第三者視点のみ | 体験談・観察を2箇所以上 |

### 文章ルール

- **文体**: ですます調で統一。「だ/である」調は禁止
- **1文**: 50文字以下
- **段落**: 3行 or 150字以内
- **体言止め**: 2-3箇所
- **H2/H3直後**: 必ずリード文を1-3行入れる

### 画像ルール

- アイキャッチ: フラットイラスト / 16:9
- 図解: 英語テキスト禁止（プロンプトに「No English text」必須）
- altテキスト必須

### 公開条件

- 5視点合計95点以上
- 95点未満で3サイクル経過 → ユーザーに差分レポートを提出して手動判断

---

## 7〜10. KPIレポート・週次最適化・スプレッドシート構成・トラブルシューティング

v1と同一内容のため、`owned-media-automation-setup.md` の Section 7〜10 を参照。

---

## カスタマイズポイント

| 箇所 | カスタマイズ内容 |
|:----|:-------------|
| .env | 全APIキー・認証情報を自分のものに置換 |
| AUTO_MODE | 初期運用は false 推奨。安定後に true へ変更 |
| 品質スコア閾値 | 95点が厳しい場合は90点に調整可 |
| CTA文言 | 自社サービス名・CTAテキストに変更 |
| E-E-A-T著者情報 | 運営者名・肩書き・実績に変更 |
| ペルソナ定義 | Phase 5の視点5のペルソナを自社ターゲットに変更 |
| 自動実行スケジュール | 記事公開ペースに合わせて調整 |
