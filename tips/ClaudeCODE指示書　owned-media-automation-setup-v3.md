# SEOオウンドメディア 全自動パイプライン構築ガイド v3

> v1・v2への依存なし。このファイル単体で完結する。
> 改善履歴: v1（原版）→ v2（AUTO_MODE・エラーハンドリング・INPUT/OUTPUT・状態管理・品質ルール集約）→ v3（Phase 2完了チェック・Phase 5採点詳細化・Phase 6完全記述・Phase 7URL修正・再開ロジック移動・Section 7〜10を内包）

---

## ⚙️ 実行モード設定

```
AUTO_MODE = false  （デフォルト）
  → Phase 3でユーザー確認待ち。確認後に次Phaseへ進む。

AUTO_MODE = true
  → Phase 3の確認をスキップ。品質95点以上で自動公開まで実行。
```

**設定の優先順位:**
```
優先度1（最高）: 実行時の $ARGUMENTS に含まれる指定
  例: /media-pipeline AUTO_MODE=true

優先度2: progress.json の "auto_mode" フィールド（前回実行時の値）

優先度3（デフォルト）: false
```

**progress.json の "auto_mode" フィールドの扱い:**
- $ARGUMENTS で指定があった場合 → progress.json に上書き記録
- $ARGUMENTS で指定がない場合 → progress.json の値を読み取って使用
- どちらもない場合 → false として動作し、progress.json に false を記録

---

## 🚨 共通エラーハンドリング（全Phaseに適用）

| エラー種別 | 対応 |
|---|---|
| API 503 / 429 | 10分間隔で最大3回リトライ → 3回失敗時はエラーログに記録してPhaseをスキップ |
| API 401 / 403 | 即停止。ユーザーに「[Phase名] 認証エラー: [API名]」を報告して終了 |
| スプレッドシートAPI障害 | `automation/logs/progress.json` にローカル記録して続行。復旧後に同期 |
| タイムアウト（30秒超） | 1回リトライ → 失敗時はスキップして次ステップへ |
| 不明なエラー | エラー内容を `automation/logs/error.log` に記録してユーザーに報告 |

---

## 📋 状態管理（progress.json）

スプレッドシートAPIが落ちていても、このファイルが最低限の状態を保証する。

```json
{
  "article_id": "042",
  "phase": "design_done",
  "kw": "Claude Code 使い方",
  "title": "",
  "published_url": "",
  "wp_post_id": null,
  "quality_score": null,
  "auto_mode": false,
  "completed_phases": ["kw_done", "research_done", "design_done"],
  "errors": [],
  "last_updated": "2026-05-09T10:00:00"
}
```

**phaseの値（全フィールドでこの表記を使う）:**
```
kw_done → research_done → design_done → write_done → quality_done → published → analyzed
```

**completed_phases との対応:**
| phase 値 | completed_phases に追加する値 |
|---|---|
| kw_done | "kw_done" |
| research_done | "research_done" |
| design_done | "design_done" |
| write_done | "write_done" |
| quality_done | "quality_done" |
| published | "published" |
| analyzed | "analyzed" |

---

## 1. 概要

### このシステムが行うこと
SEOオウンドメディアの記事制作を、KW選定から公開後処理まで全自動で実行する。

### 自動化される工程
1. **Phase 1 / KW選定**: Ahrefs APIで競合KW抽出→3C分析→SEO Knowledge批判3周→確定
2. **Phase 2 / リサーチ**: X API v2で関連投稿30件+YouTube文字起こし5-10本
3. **Phase 3 / 設計**: 検索上位10記事のH2構造分析→共通60-70%+独自30-40%の構成確定
4. **Phase 4 / 執筆**: 80%ドラフト→一次情報20%組み込み→AI感排除→品質批判10-20周
5. **Phase 5 / 品質**: 5視点採点（95点以上が公開条件）
6. **Phase 6 / 公開**: アイキャッチ+H2直下図解→WordPress自動投稿→公開
7. **Phase 7 / 分析**: インデックス登録→スプレッドシート更新→内部リンク追加→KPIレポート

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
| Google Indexing API | 公開後の即時インデックス登録 | サービスアカウント必須。GSCオーナー権限設定が必要 |
| AI画像生成ツール（NanoBanana MCP or Gemini API） | 画像生成 | NanoBanana推奨。Gemini Flash/Proでも可 |
| yt-dlp | YouTube自動字幕取得 | `pip install yt-dlp` |
| Python 3.10+ | Markdown→HTML変換、スクリプト実行 | `markdown` ライブラリ（extensions: tables, extra） |

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
│   │   ├── progress.json
│   │   └── error.log
│   └── design/
├── data/
│   ├── x_trends/
│   └── youtube_transcripts/
├── kpi_feedback.md
└── wp_additional_css.css
```

```bash
mkdir -p articles automation/{images,logs,design} data/{x_trends,youtube_transcripts}
echo '{"article_id":"","phase":"","kw":"","title":"","published_url":"","wp_post_id":null,"quality_score":null,"auto_mode":false,"completed_phases":[],"errors":[],"last_updated":""}' > automation/logs/progress.json
touch .env kpi_feedback.md automation/logs/error.log
```

### 3.2 スプレッドシートの準備

Google Sheetsで新規スプレッドシートを作成し、以下の10タブを用意する（詳細は Section 9 参照）:
`KW戦略 / トピッククラスター / X一次情報 / YouTube一次情報 / ダッシュボード / 記事作成ログ / KPIレポート / 内部リンク管理 / リライトログ / エラーログ`

### 3.3 WordPress設定

1. REST API有効化（WordPress 4.7以降はデフォルトで有効）
2. Application Password作成: ユーザー設定 → Application Passwords → 新規作成
3. Rank Mathインストール・有効化
4. パーマリンク設定: `/%category%/%postname%/` 推奨
5. カテゴリ作成してIDを控える

### 3.4 CLAUDE.mdへの追記

```markdown
## オウンドメディア自動パイプライン
- 「記事を書いて」「パイプライン実行」→ Phase 1〜7 を順番に実行
- 「KPI確認」→ Phase 7 の KPIレポート生成を実行
- 「リライト」→ Section 8 の週次最適化を実行
```

---

## 4. 再開ロジック（パイプライン開始時に必ず確認）

パイプライン開始時に `automation/logs/progress.json` を読み込み、以下のルールで開始Phaseを決定する。

```
progress.json の phase 値 → 開始するPhase
─────────────────────────────────────────
""（空）または "analyzed"  → Phase 1 から新規開始
"kw_done"                  → Phase 2 から再開
"research_done"            → Phase 3 から再開
"design_done"              → Phase 4 から再開
"write_done"               → Phase 5 から再開
"quality_done"             → Phase 6 から再開
"published"                → Phase 7 から再開
```

再開時の確認手順:
1. 再開Phaseの【インプット】ファイルが存在するか確認する
2. 存在しない場合は1つ前のPhaseから再実行する
3. `errors` フィールドにエラーが残っている場合はユーザーに内容を報告してから再開する

**強制リセット（最初からやり直す場合）:**
```bash
echo '{"article_id":"","phase":"","kw":"","title":"","published_url":"","wp_post_id":null,"quality_score":null,"auto_mode":false,"completed_phases":[],"errors":[],"last_updated":""}' > automation/logs/progress.json
```

---

## 5. 自動化パイプライン定義（全7Phase）

> **共通ルール**
> - 各Phaseの開始時に `automation/logs/progress.json` を読み込む
> - 各Phaseの完了時に `progress.json` を更新してから次に進む
> - エラー発生時は冒頭「共通エラーハンドリング」セクションに従う

---

### Phase 1: KW選定

```
【インプット】
- automation/logs/progress.json
- スプレッドシート「記事作成ログ」タブの未着手行（障害時はスキップ可）

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
   - スプレッドシート更新（障害時はスキップして後で同期）

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

【完了条件チェック】
以下のコマンドで件数を確認し、基準を満たしていることを確認してから次に進む:
```bash
# X投稿: 30件以上
ls data/x_trends/x_enriched_article*.json 2>/dev/null | wc -l
# → 30 以上であること

# YouTube文字起こし: 5本以上
ls data/youtube_transcripts/*.txt 2>/dev/null | wc -l
# → 5 以上であること
```
```

**目的**: 記事に独自性を持たせるための一次情報を収集する。

**実行手順**:

1. **X一次情報の収集**
   - X API v2で記事KWに関連する投稿を検索（直近1ヶ月・min_impressions: 500）
   - 必ず取得するフィールド: `note_tweet`（長文全文）/ スレッド全文 / メディアURL / 投稿URL
   - 30件以上を目標に収集して `data/x_trends/x_enriched_article{N}.json` に保存
   - スプレッドシート「X一次情報」タブに記録（G列は200字以上の全文を記載）

2. **YouTube一次情報の収集**
   - YouTube Data API v3で関連動画を検索（上位10本）
   - `yt-dlp` で自動字幕を取得:
     ```bash
     yt-dlp --write-auto-sub --sub-lang ja --skip-download \
       -o "data/youtube_transcripts/%(id)s" "VIDEO_URL"
     ```
   - スプレッドシート「YouTube一次情報」タブに記録
   - **注意**: H列（文字起こし完了）が全件「○」になるまで次に進まない

3. **完了確認・状態更新**
   - 上記【完了条件チェック】のコマンドで件数を確認
   - 基準未満の場合はAPIを再試行する（共通エラーハンドリングに従う）
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
- articles/draft_{article_id}_structure.md が存在すること
- AUTO_MODE=false の場合: ユーザー承認済みであること
- progress.json の phase が "design_done" になっていること
```

**目的**: 検索上位記事の構造を分析し、共通パターン+独自パートで記事構成を確定する。

**実行手順**:

1. **検索上位10記事のH2/H3構造抽出**
   - Ahrefs `serp-overview` で検索上位10記事のURL / DR / タイトルを取得
   - 全10サイト（最低7サイト）のページをフェッチし、H2/H3見出しを抽出

2. **共通パターン分析**
   - **共通60-70%**: 出現率50%以上のH2は踏襲（検索意図の核心）
   - **独自30-40%**: 出現率25%以下 or 0%のH2を一次情報から設計（差別化ポイント）

3. **一次情報の配置確定**
   - 確定H2ごとに「使うX投稿」「使うYouTube動画」を具体的に決定
   - メディア（画像/動画）の記事埋め込み可否も判断

4. **⚠️ ユーザー確認（AUTO_MODEに従う）**
   ```
   AUTO_MODE=false: 確定H2構成を提示 → ユーザーの「承認」入力を待つ → Phase 4へ
   AUTO_MODE=true:  H2構成を記録して自動的に Phase 4へ
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
- articles/draft_{article_id}.md が存在すること
- 文字数 5,000字以上
- AI感排除7項目チェックが全て ✅ であること（Section 6「品質ルール」参照）
- progress.json の phase が "write_done" になっていること
```

**目的**: 確定した構成に基づいて高品質な記事本文を生成する。

**実行手順**:

1. **80%ドラフト生成**
   - 確定H2構成に沿って本文を生成
   - 各H2に紐付けた一次情報を自然に組み込む（全体の20%以上）
   - 独自ファクト3箇所以上を含める / 目標文字数: 5,000字以上

2. **AI感排除チェック（7項目）**
   → Section 6「品質ルール」の AI感排除7項目を参照して全項目チェック・修正

3. **文章ルール適用**
   → Section 6「品質ルール」の文章ルールを参照して適用

4. **X引用スタイル適用**
   - 自分の体験・感想を先に述べ、後から「〜氏も同様のことを言っています」と軽くリンクを添える
   - blockquoteブロック使用禁止。インラインで自然に組み込む
   - @username は `<a href="https://x.com/username" target="_blank" rel="noopener">@username</a>` 形式

5. **SEO Knowledge品質批判10-20周**
   - 検索意図との整合性 / E-E-A-T要素の充実度 / 内部リンク候補の配置 / 独自性 / 読者が行動できるか

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
- articles/final_{article_id}.md が存在すること
- progress.json の phase が "quality_done" になっていること
```

> ⚠️ **役割切り替え（重要）**
> このPhaseに入る前に必ず宣言する:
> 「あなたはPhase 4で記事を書いたエージェントとは完全に別の存在です。
> 今から厳格な編集長として振る舞い、この記事を初めて読む視点で採点してください。
> 執筆プロセスへの配慮は一切不要。読者の利益のみを基準に採点すること。」

**目的**: SEO最適化と多角的品質チェックで95点以上を達成する。

**実行手順**:

1. **メタ情報設定**
   - タイトル: 30文字以内、KWを先頭に配置、数字を含める
   - メタディスクリプション: 120文字以内、検索意図を満たす文章
   - 記事冒頭にKWを含む太字結論文を配置

2. **E-E-A-T注入**
   - 著者情報: 運営者名・肩書き・実績を記載
   - データ引用: Ahrefs / 公的統計 / 権威ある情報源からの引用
   - 事例: 自社サービスの導入企業・成功事例

3. **内部リンク・外部リンク**
   - クラスター内記事への内部リンク: 3-5本
   - 外部権威ソースへのリンク: 3-5本（公式サイト・公的機関等）

4. **GEO/LLMO構造対応**
   - 各H2冒頭に1文結論を設置（AI Overviewが引用しやすい構造）
   - Citation形式: 定義ブロック→数値ファクト3箇所→比較テーブル→Q&A形式
   - FAQ Schema 5問以上

5. **5視点採点（各20点 / 合計100点）**

   **視点1: デザイン（フォーマット）【20点】**
   - H2構成が目次に対応し、各H2冒頭に1文結論があるか（2点）
   - テーブルのヘッダー・スタイルが統一されているか（2点）
   - CTAが2箇所以上で配置されているか（2点）
   - H2直下に図解画像が配置されているか（2点）
   - テキスト強調（strong）が12-18箇所に適切に配置されているか（2点）
   - 段落が3行以内で読みやすいか（2点）
   - 箇条書きの前後にリード文があるか（2点）
   - プロンプト例がコードブロック形式で掲載されているか（2点）
   - 画像にaltテキストが設定されているか（2点）
   - 全体のフォーマットが統一されているか（2点）

   **視点2: SEO【20点】**
   - タイトルにKWが自然に含まれているか・30文字以内か（2点）
   - メタディスクリプションが120文字以内で検索意図を満たすか（2点）
   - 本文が5,000文字以上あるか（2点）
   - 各H2冒頭に1文結論（GEO対応）があるか（2点）
   - 外部リンクが権威あるソースに設定されているか（2点）
   - E-E-A-T要素（著者実績・数値・一次情報）が含まれているか（2点）
   - FAQセクションが5問以上あるか（2点）
   - 内部リンクが3本以上あるか（2点）
   - KWの検索意図と記事構成が一致しているか（2点）
   - 独自ファクトが3箇所以上あるか（2点）

   **視点3: 編集（文体）【20点】**
   - AI感のある表現が排除されているか（7項目チェック）（2点）
   - ですます調で統一されているか（2点）
   - リード文が2-3文で簡潔か（2点）
   - 一人称の体験談・観察が2箇所以上あるか（2点）
   - X/YouTube引用がインラインで自然に組み込まれているか（2点）
   - 専門用語が初出時に平易に説明されているか（2点）
   - 1文50字以下・段落3行以内が守られているか（2点）
   - 文末パターンにバリエーションがあるか（2点）
   - 体言止めが2-3箇所あるか（2点）
   - 「この記事でわかること」ボックスが冒頭にあるか（2点）

   **視点4: 技術正確性【20点】**
   - ツールの説明が正確で最新か（2点）
   - プロンプト例が実用的か（コピーして即使用可能）（2点）
   - ワークフロー手順が初心者でも再現できる粒度か（2点）
   - 専門概念が平易に説明されているか（2点）
   - ツールの限界・注意点が正確に記載されているか（2点）
   - 導入事例の数値が具体的で信頼性があるか（2点）
   - 比較情報が正確か（料金・機能・対応状況）（2点）
   - 「次に試すべきアクション」が具体的か（2点）
   - 最新動向と記事内容に齟齬がないか（2点）
   - ユースケースが具体的か（業務・入力・出力）（2点）

   **視点5: 読者UX（ペルソナ: 中小企業経営者・ITリテラシー中程度）【20点】**
   - リード文で「自分の悩みに答えてくれる」と感じられるか（2点）
   - 専門用語が初出時に説明されているか（2点）
   - 各H2の結論を読むだけで得られるものが分かるか（2点）
   - 導入事例が「自分でも再現できそう」と感じられるか（2点）
   - 「まず何から始めるか」が明確か（2点）
   - 段落が3-4文以内で読みやすいか（2点）
   - 表が単体で意味が完結しているか（2点）
   - FAQが実際にペルソナが抱く疑問と一致しているか（2点）
   - CTAが「申し込みたくなる」文脈で提示されているか（2点）
   - 読後に「次に何をすればいいか」が分かるか（2点）

6. **修正ループ**
   ```
   採点 → 合計 < 95点:
     1. 最低スコアの視点の指摘を優先修正
     2. 修正した視点のみ再採点（他のスコアは維持）
     3. 最大3サイクル繰り返す
     4. 3サイクル後も95点未満 → ユーザーに差分レポートを提出して停止
   ```

7. **状態更新**
   - `articles/final_{article_id}.md` に最終稿を保存
   - `progress.json` を更新（quality_score に合計点を記録）

---

### Phase 6: 公開

```
【インプット】
- progress.json: { "phase": "quality_done", "quality_score": 95以上 }
- articles/final_{article_id}.md（最終稿）

【アウトプット】
- WordPressに記事が公開済み
- progress.json 更新: { "phase": "published", "wp_post_id": 1234, "published_url": "https://..." }

【完了条件】
- WordPress REST APIでステータスが "publish" になっていること
- wp_post_id と published_url が progress.json に記録されていること
- AUTO_MODE=false の場合: ユーザーが下書き確認後に公開指示を出したこと
```

**目的**: アイキャッチ画像と記事内図解を生成し、WordPressに公開する。

**実行手順**:

1. **アイキャッチ画像生成**
   - NanoBanana Flash または Gemini Flash で生成（16:9 / フラットイラスト）
   - フォールバック順: NanoBanana Pro → NanoBanana Flash → Gemini Flash
   - 記事タイトルから最重要KWを抽出してデザインに反映

2. **H2直下の図解画像生成**
   - プロンプトに必ず付与: `No English text whatsoever. Japanese text only or no text at all.`
   - H2の中で視覚化が効果的なものを選んで生成（目安: 3-5枚）
   - 生成後に英語テキスト混入がないか目視確認。混入があれば再生成

3. **Markdown→HTML変換**
   ```bash
   python3 -c "
   import markdown
   with open('articles/final_{article_id}.md') as f:
       md = f.read()
   html = markdown.markdown(md, extensions=['tables','extra'])
   print(html)
   " > automation/images/article_{article_id}.html
   ```
   - Gutenbergブロック（`<!-- wp:buttons -->` 等）は変換前に退避し、変換後に復元
   - 画像プレースホルダーは `<figure>` タグに置換

4. **WordPress投稿（下書き）**
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/posts" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "記事タイトル",
       "content": "HTML本文",
       "status": "draft",
       "categories": [カテゴリID],
       "slug": "url-slug",
       "excerpt": "メタディスクリプション"
     }'
   # レスポンスから "id" を取得して wp_post_id に記録
   ```

5. **画像アップロード**
   ```bash
   curl -X POST "${WP_URL}/wp-json/wp/v2/media" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Disposition: attachment; filename=image.png" \
     --data-binary @automation/images/eyecatch_{article_id}.png
   # レスポンスの "id" を取得して featured_media に設定
   ```
   - アイキャッチはPOST更新で `featured_media` に設定
   - 記事内画像はHTMLの `<figure>` タグで参照

6. **Rank Math SEO設定**
   ```bash
   curl -X POST "${WP_URL}/wp-json/rankmath/v1/updateMeta" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{
       "objectID": WP_POST_ID,
       "objectType": "post",
       "meta": {
         "rank_math_title": "SEOタイトル（KW含む・30文字以内）",
         "rank_math_description": "メタディスクリプション（120文字以内）",
         "rank_math_focus_keyword": "フォーカスKW",
         "rank_math_robots": ["index", "follow"]
       }
     }'
   ```

7. **FAQ Schema設定**
   ```bash
   curl -X POST "${WP_URL}/wp-json/rankmath/v1/updateSchemas" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{
       "objectID": WP_POST_ID,
       "objectType": "post",
       "schemas": {
         "schema-FAQPage-1": {
           "schema-type": "FAQPage",
           "schema-name": "FAQPage-1",
           "schema-data": {
             "@type": "FAQPage",
             "mainEntity": [
               {"@type": "Question", "name": "Q1", "acceptedAnswer": {"@type": "Answer", "text": "A1"}}
             ]
           }
         }
       }
     }'
   ```

8. **公開（⚠️ AUTO_MODEに従う）**
   ```bash
   # AUTO_MODE=false: 下書きURLをユーザーに提示 → 「公開」入力を待つ
   # AUTO_MODE=true:  品質スコア95点以上を確認して以下を実行
   curl -X POST "${WP_URL}/wp-json/wp/v2/posts/${WP_POST_ID}" \
     -u "${WP_USER}:${WP_PASSWORD}" \
     -H "Content-Type: application/json" \
     -d '{"status": "publish"}'
   ```

9. **状態更新**
   ```json
   { "phase": "published", "wp_post_id": 1234, "published_url": "https://YOUR_DOMAIN.com/slug/" }
   ```

---

### Phase 7: 分析

```
【インプット】
- progress.json: { "phase": "published", "wp_post_id": 1234, "published_url": "https://..." }

【アウトプット】
- Googleインデックス登録完了
- スプレッドシート全タブ更新
- kpi_feedback.md 更新
- progress.json 更新: { "phase": "analyzed" }

【完了条件】
- Indexing APIへのリクエストが成功していること（または GSC手動登録依頼済み）
- progress.json の phase が "analyzed" になっていること
```

**目的**: インデックス登録、スプレッドシート更新、内部リンク追加、KPIレポート生成。

**実行手順**:

1. **Google Indexing APIでインデックス登録**
   ```python
   import json
   from google.oauth2 import service_account
   from googleapiclient.discovery import build

   # progress.json から published_url を取得
   with open('automation/logs/progress.json') as f:
       progress = json.load(f)
   published_url = progress['published_url']

   credentials = service_account.Credentials.from_service_account_file(
       'indexing-service-account.json',
       scopes=['https://www.googleapis.com/auth/indexing']
   )
   service = build('indexing', 'v3', credentials=credentials)
   service.urlNotifications().publish(
       body={'url': published_url, 'type': 'URL_UPDATED'}
   ).execute()
   ```
   - 失敗時: `errors` に記録してユーザーにGSC手動登録を依頼（スキップして続行）

2. **スプレッドシート全タブ更新（共通エラーハンドリング適用）**

   | タブ | 更新内容 |
   |:----|:--------|
   | ダッシュボード | 最終更新日・総公開記事数・今月公開数・インデックス済数 |
   | 記事作成ログ | ステータスを「公開済み」に更新。URL・品質スコア記録 |
   | KW戦略 | 該当KWのStatusを「公開済」に更新 |
   | トピッククラスター | 該当KWのStatusを「公開済」に更新。URL追加 |
   | KPIレポート | 新行追加（日付・総記事数・本日公開数・インデックス率） |

   - スプレッドシートAPI障害時: `progress.json` の `errors` に記録して続行。後で手動同期

3. **既存記事への内部リンク自動追加**
   - 公開済み全記事のH2見出しとKWを取得
   - 新規記事のKWが自然に挿入できる箇所を特定
   - WordPress REST APIで既存記事を更新して内部リンクを追加
   - 「内部リンク管理」タブに記録

4. **kpi_feedback.md の更新**
   - 成功パターン・失敗パターン・リライト優先度を追記
   - 翌朝のパイプラインが読み込んで品質改善に活用

5. **状態更新**
   - `progress.json` を `{ "phase": "analyzed" }` に更新
   - `errors` フィールドに残ったエラーがあればユーザーに最終レポートとして提示

---

## 6. 自動スケジュール定義

| スケジュール | 実行時刻(JST) | 内容 |
|:----------|:------------|:----|
| Morning Pipeline | 毎日 5:00 | Phase 1〜7 / AUTO_MODE=true |
| Afternoon Pipeline | 毎日 14:00 | Phase 1〜7 / AUTO_MODE=true |
| Daily KPI Report | 毎日 22:13 | Phase 7のKPIレポートのみ |
| Weekly Optimize | 月曜 10:23 | Section 8 の週次最適化を実行 |

### パイプライン実行フロー
```
1. kpi_feedback.md を読み込む（前日の成功/失敗パターンを反映）
2. progress.json を読み込み、Section 4「再開ロジック」で開始Phaseを決定
3. AUTO_MODE を確定（$ARGUMENTS → progress.json → デフォルトfalse の優先順）
4. 決定したPhaseから順番に実行
5. 各Phase完了後に progress.json を更新
6. 全Phase完了後に完了通知
```

---

## 7. 品質ルール（単一参照元）

> ⚠️ Phase 4 と Phase 5 の両方がこのセクションを参照する。ルール変更はここだけ修正すること。

### AI感排除7項目（全記事で必ずチェック）

| # | チェック項目 | NG例 | OK例 |
|:--|:-----------|:----|:----|
| 1 | あいまいな結論 | 「場合もあります」「可能性があります」 | 具体的シナリオ・条件を明示 |
| 2 | 重要ですの連発 | 「〜することが重要です」が4回以上 | 記事中3回まで。代わりに具体的アクションを提示 |
| 3 | 文末パターン同一 | 全段落が「〜です。」で終わる | 「〜です」「〜ます」「〜でしょう」「体言止め」を交互に |
| 4 | 3連続箇条書き | 箇条書き→箇条書き→箇条書き | 間にリード文やテーブルを挟む |
| 5 | これにより | 「これにより効率が向上します」 | 完全削除。具体的な因果関係を文章で記述 |
| 6 | おすすめします | 「検討することをおすすめします」 | 「してください」「試してみてください」 |
| 7 | 一人称の欠如 | 第三者視点のみの記事 | 一人称の観察・体験談を2箇所以上 |

### 文章ルール

- **文体**: ですます調で統一。「だ/である/〜した」調は一切禁止
- **1文**: 50文字以下
- **段落**: 3行 or 150字以内
- **文長バリエーション**: 短文と長文を交互に
- **体言止め**: 2-3箇所
- **H2/H3直後**: 必ずリード文を1-3行入れる（いきなり表やリストに入らない）
- **H4見出し**: サブステップには `<strong>` ではなくH4タグを使用
- **プロンプト表示**: コードブロック形式で掲載

### 記事構成共通ルール

- **目次**: WordPressテーマのTOC機能が自動生成。手動挿入しない
- **テキスト強調**: `<strong>` タグで太字（12-18箇所）。核心フレーズ・数値データ・アクション喚起文に限定
- **CTA配置**: 記事内に最低2箇所。中央揃えボタンブロックで統一
- **「この記事でわかること」ボックス**: 冒頭に必ず設置（ポゴスティッキング防止）
- **FAQ**: 5問以上。FAQPage Schemaを設定

### 画像ルール

- **アイキャッチ**: フラットイラスト / 16:9 / カテゴリカラー準拠
- **H2直下図解**: 英語テキスト禁止（プロンプトに「No English text」必須）
- **altテキスト必須**: 画像内容を簡潔に記述。KW詰め込みNG

### 公開条件

- 5視点合計95点以上
- 95点未満で3サイクル経過 → ユーザーに差分レポートを提出して手動判断

---

## 8. KPIレポート定義

### 取得データ

| データソース | 取得項目 |
|:----------|:--------|
| WordPress REST API | 公開記事数、draft記事数、カテゴリ別記事数 |
| GA4 Data API | PV、エンゲージメント時間、直帰率、ページ/セッション |
| GSC Data API | 表示回数、クリック数、CTR、平均順位 |
| Ahrefs API | DR、被リンク数、オーガニックKW数、推定トラフィック |
| サイトマップ | 登録URL数、最終更新日 |

### 成功/失敗パターンの抽出ロジック

**成功パターン（kpi_feedback.mdの「成功パターン」セクションに追記）:**
- 公開7日以内にGSCで表示回数が発生した記事 → その構造・KW特性を記録
- 品質スコア97点以上の記事 → その特徴（文字数・一次情報数・Schema種類）を記録
- クラスター完成後に全体順位が上昇したケース → クラスター効果を記録

**失敗パターン（kpi_feedback.mdの「失敗パターン」セクションに追記）:**
- Vol=200以下のKWで記事を作成したケース → 月間推定セッション上限が低い
- インデックスされていない記事 → サイトマップ・Schema・内部リンクの問題を特定
- カニバリが発生したKW → 対処法（統合 or 差別化）を記録

### kpi_feedback.md の生成フォーマット

```markdown
# KPIフィードバック（自動更新: YYYY-MM-DD）

## サイト概況
| 指標 | 値 | 前日比 | 備考 |
|:----|:--|:------|:----|
| 累計公開記事数 | N本 | +N | |
| ドメインレーティング | N | +N | |
| オーガニックKW数 | N | +N | |
| 推定オーガニックトラフィック | N | +N | |

## 今日の主な動き
1. ...

## 成功パターン（これを踏襲せよ）
- ...

## 失敗パターン（これを避けよ）
- ...

## リライト優先度リスト
| 記事 | 順位 | 改善ポイント | 期待効果 |
```

---

## 9. 週次最適化定義（Weekly Optimize）

毎週月曜 10:23 に実行。

### 9.1 カニバリ検出ロジック

1. GSC Data APIで全記事の「ランクインKW」を取得
2. 同一KWで2本以上の記事がランクインしているケースを抽出
3. カニバリ発生時の対処:
   - **統合**: 品質の低い方を削除し、高い方にコンテンツを統合
   - **差別化**: H1・メタディスクリプション・記事構成を明確に差別化
   - **301リダイレクト**: 統合した場合は旧URLから新URLへリダイレクト

### 9.2 リライト対象の選定基準

| 優先度 | 条件 | 対処 |
|:------|:----|:----|
| 最高 | GSC順位11-30位（2-3ページ目） | H2追加・一次情報追加・内部リンク強化で1ページ目に引き上げ |
| 高 | 表示回数高×CTR低 | タイトル・メタディスクリプション改善 |
| 中 | 公開30日以上経過 & インデックス未登録 | テクニカルSEO確認（サイトマップ・Schema・内部リンク） |
| 低 | 情報が古くなった記事 | 料金・機能・バージョン情報の更新 |

### 9.3 内部リンク最適化

1. 全公開記事の内部リンク数を確認
2. 内部リンクが3本未満の記事 → 関連記事への内部リンクを追加
3. 新規クラスター記事が公開された場合 → 同一クラスター内の全記事に相互リンクを追加
4. 「内部リンク管理」タブを更新

### 9.4 テクニカルSEO監視項目

| 項目 | チェック方法 | 期待値 |
|:----|:----------|:------|
| サイトマップ | post-sitemap.xml をフェッチ → URL数を確認 | 公開記事数と一致 |
| robots.txt | robots.txt をフェッチ → ブロック状況確認 | LLMクローラー許可済み |
| Core Web Vitals | PageSpeed Insights API | LCP<=2.5s, INP<=200ms, CLS<=0.1 |
| HTTPS | SSL証明書の有効期限 | 有効期限30日以上 |
| 404ページ | GSCのカバレッジレポート | 404ゼロ |
| Schema | 各記事のJSON-LD確認 | BlogPosting+FAQPage+BreadcrumbList |

---

## 10. スプレッドシート構成（詳細）

### KW戦略タブ
```
A: KW / B: Volume / C: KD / D: Intent（info/commercial/transactional）
E: CV距離（Do > Buy > Commercial > Info の順でスコアリング）
F: 自社独自価値 / G: Status（未着手/執筆中/公開済/リライト中） / H: 備考
```

### トピッククラスタータブ
```
A: KW / B: Volume / C: KD / D: カテゴリ / E: クラスター名
F: 種別（ピラー/クラスター） / G: 優先度（A/B/C）
H: Status（未着手/公開済） / I: URL（公開後に記入）
```

### X一次情報タブ
```
A: 記事# / B: KW / C: 投稿者名（@username） / D: 投稿日 / E: 投稿URL
F: 本文（note_tweet全文。200字以上） / G: インプレッション数
H: RT数 / I: Like数 / J: メディアURL / K: 記事への活用方法
```
**注意**: 記事間に空行を入れない。#1→#2→#3と隙間なく詰めて記録する。

### YouTube一次情報タブ
```
A: 記事# / B: KW / C: 動画タイトル / D: チャンネル名 / E: 動画URL
F: 再生数 / G: 文字起こしファイルパス / H: 文字起こし完了（○/×）
I: 要約 / J: 記事への活用方法
```
**注意**: H列が全件「○」になるまでPhase 4に進まない。

### ダッシュボードタブ
```
A: 指標名 / B: 値 / C: 前日比 / D: 備考
```
主な指標: 累計公開記事数 / DR / 被リンク数 / オーガニックKW数 / 推定トラフィック / インデックス済み記事数 / 今月公開数

### 記事作成ログタブ
```
A: 記事# / B: 制作日 / C: KW / D: タイトル / E: URL / F: カテゴリ / G: Volume
H: 品質ループ回数 / I: 品質スコア / J: ステータス / K: WP Post ID / L: 備考
```
**注意**: パイプラインは必ずこのタブの「未着手」行の一番上から順番に開始する。

### KPIレポートタブ
```
A: 日付 / B: 総公開記事数 / C: 本日公開数 / D: インデックス率（%）
E: PV（GA4） / F: 表示回数（GSC） / G: クリック数（GSC）
H: CTR（GSC） / I: 平均順位（GSC） / J: 備考
```

### 内部リンク管理タブ
```
A: リンク元記事 / B: リンク先記事 / C: アンカーテキスト / D: 設置日 / E: リンク先URL
```

### リライトログタブ
```
A: 記事# / B: リライト日 / C: リライト理由 / D: 変更箇所の概要
E: リライト前順位 / F: リライト後順位（1週間後に記入） / G: 効果（改善/変化なし/悪化）
```

### エラーログタブ
```
A: 日付 / B: Phase（1-7） / C: エラー内容 / D: 対応内容
E: 解決状態（解決済み/未解決/回避策適用）
```

---

## 11. トラブルシューティング

### API 503エラー時の対応
```
1. 即フォールバックしない
2. 10分間隔でポーリング（最大6回 = 60分）
3. 6回連続503の場合:
   - エラーログタブに記録
   - 該当Phaseをスキップして次に進む
   - または翌日のパイプラインで再実行
```

### 画像生成失敗時のフォールバック
```
1. NanoBanana Pro → NanoBanana Flash → Gemini Flash の順でフォールバック
2. 全て失敗した場合:
   - 画像なしで記事を公開（altテキストは設定）
   - 翌日に画像を再生成して追加アップロード
3. 日本語文字化けの場合:
   - 英語テキストのみで再生成
   - 日本語はキャプション（figcaption）で補完
```

### WP投稿エラー時の対応
```
1. 401 Unauthorized → Application Passwordの再生成
2. 403 Forbidden → ユーザーの権限確認（Administrator or Editor）
3. 500 Internal Server Error → WP管理画面でエラーログ確認
4. 記事HTMLが崩れる → Markdown→HTML変換を再確認（Gutenbergブロック退避・復元）
5. カテゴリIDエラー → GET /wp/v2/categories で正しいIDを確認
```

### スプレッドシートトークン期限切れ
```
1. Google Sheets APIのOAuthトークンは通常7日で期限切れ
2. credentials.json からリフレッシュトークンで再取得
3. MCP経由の場合はMCPサーバーの再認証を実行
4. sheets-token.json の更新日を確認し、7日以上前なら更新
```

### Ahrefs API units枯渇
```
1. 月次リセット日を確認（通常は月の同一日付）
2. リセットまでの期間はAhrefsクエリを一切使用しない
3. KW選定は記事作成ログの未着手行から取得（Ahrefsリサーチ不要）
4. リセット後に実行すべきクエリの優先順位:
   a. site-explorer-metrics（全体把握）
   b. site-explorer-organic-keywords（全記事のランクインKW）
   c. site-explorer-top-pages（トラフィック順TOP記事）
   d. rank-tracker-overview（追跡KW順位）
```

### インデックス登録が進まない場合
```
1. サイトマップ確認: post-sitemap.xml をフェッチしてURL数を確認
2. Rank Math設定: サイトマップに全記事が含まれているか確認
3. Indexing APIのサービスアカウントがGSCオーナー権限を持っているか確認
4. robots.txt でクローラーをブロックしていないか確認
5. 手動対応: GSC URL検査ツールで1記事ずつ手動登録
```

---

## カスタマイズポイント

| 箇所 | カスタマイズ内容 |
|:----|:-------------|
| .env | 全APIキー・認証情報を自分のものに置換 |
| AUTO_MODE | 初期運用は false 推奨。安定後に true へ変更 |
| 品質スコア閾値 | 95点が厳しい場合は90点に調整可（推奨は95点以上） |
| CTA文言 | 自社サービス名・CTAテキストに変更 |
| E-E-A-T著者情報 | 運営者名・肩書き・実績に変更 |
| カテゴリ | 自社メディアのカテゴリ構成に変更 |
| ペルソナ定義 | Phase 5の視点5のペルソナを自社ターゲットに変更 |
| 自動実行スケジュール | 記事公開ペースに合わせて調整 |
| 画像スタイル | ブランドカラー・デザインガイドラインに合わせて変更 |
