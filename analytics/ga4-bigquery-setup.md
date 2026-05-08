# GA4 × BigQuery 連携セットアップガイド

対象: Google Analytics 4（GA4）× BigQuery によるユーザー行動分析  
前提: iOSアプリ（Flutter）のリリース後分析を想定

---

## 全体の流れ

```
[iOSアプリ] → Firebase SDK → [GA4プロパティ]
                                    ↓ BigQuery Export（自動）
                             [BigQuery データセット]
                                    ↓ SQL クエリ
                             [ユーザー行動分析]
```

---

## 事前に必要なもの

| 項目 | 内容 |
|---|---|
| GA4プロパティ | Firebase プロジェクトに紐づいたもの |
| Google Cloudプロジェクト | BigQueryを有効化するGCPプロジェクト |
| 権限 | GCPプロジェクトの編集者以上 / GA4の編集者以上 |

---

## Step 1: Google Cloudプロジェクトの準備

### 1-1. プロジェクト作成（既存があればスキップ）

1. [Google Cloud Console](https://console.cloud.google.com/) を開く
2. 上部の「プロジェクト選択」→「新しいプロジェクト」
3. プロジェクト名を入力（例: `sakatsu-log-analytics`）→「作成」

### 1-2. BigQuery APIを有効化

```
Cloud Console → APIとサービス → ライブラリ → 「BigQuery API」を検索 → 有効にする
```

または Cloud Shell で:

```bash
gcloud services enable bigquery.googleapis.com
```

### 1-3. 請求アカウントを紐づける

BigQueryの無料枠を超えた場合に課金されるため、請求アカウントを設定しておく。

```
Cloud Console → 請求 → 請求先アカウントをリンク
```

**無料枠の目安（個人開発規模では超えにくい）:**
- ストレージ: 毎月10GBまで無料
- クエリ: 毎月1TBまで無料

---

## Step 2: GA4 と BigQuery を連携する

### 2-1. GA4管理画面で設定

1. [GA4 管理画面](https://analytics.google.com/) を開く
2. 左下「管理」→ プロパティ列の「**BigQuery のリンク**」
3. 「リンク」ボタンをクリック
4. BigQueryプロジェクトを選択
5. データセットのロケーション: **`asia-northeast1`（東京）** を選択
6. エクスポートの種類を選択:

| エクスポート種別 | 更新頻度 | コスト | 推奨用途 |
|---|---|---|---|
| **毎日** | 1日1回（翌日） | 無料 | 個人開発・MVP段階 |
| ストリーミング | リアルタイム | 有料 | 本番・大規模 |

7. 「送信」で完了

### 2-2. エクスポート確認（翌日以降）

BigQuery のコンソール（[console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)）で以下が作成される:

```
プロジェクト名
└── analytics_XXXXXXXXX（GA4プロパティID）
    ├── events_20260508    ← 日付ごとのテーブル
    ├── events_20260509
    └── events_intraday_20260508  ← 当日分（ストリーミング時のみ）
```

---

## Step 3: データスキーマを理解する

GA4のエクスポートデータは **`events_YYYYMMDD`** テーブルに格納される。

### 主要カラム

| カラム名 | 内容 | 例 |
|---|---|---|
| `event_date` | イベント発生日 | `20260508` |
| `event_name` | イベント名 | `screen_view`, `session_start` |
| `event_params` | イベントのパラメータ（配列） | `[{key: "screen_name", value: {string_value: "home"}}]` |
| `user_pseudo_id` | 匿名ユーザーID（端末ごと） | `abc123...` |
| `user_id` | ログイン済みユーザーID | `user_001`（設定した場合） |
| `platform` | プラットフォーム | `iOS` |
| `geo.country` | 国 | `Japan` |
| `app_info.version` | アプリバージョン | `1.0.0` |

### event_params の取り出し方（重要）

`event_params` は配列のため、SQLで取り出す際は `UNNEST` または `(SELECT ...)` を使う:

```sql
-- screen_name パラメータを取り出す例
SELECT
  event_name,
  (SELECT value.string_value
   FROM UNNEST(event_params)
   WHERE key = 'screen_name') AS screen_name
FROM `プロジェクト名.analytics_XXXXXXXXX.events_20260508`
```

---

## Step 4: よく使うクエリ集

以下のクエリは `プロジェクト名.analytics_XXXXXXXXX` を実際の値に置き換えて使用する。

### 4-1. 日別アクティブユーザー数（DAU）

```sql
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS dau
FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260501' AND '20260531'
GROUP BY event_date
ORDER BY event_date
```

### 4-2. 画面別ビュー数（スクリーンビュー）

```sql
SELECT
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'screen_name') AS screen_name,
  COUNT(*) AS view_count
FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260501' AND '20260531'
  AND event_name = 'screen_view'
GROUP BY screen_name
ORDER BY view_count DESC
```

### 4-3. 記録ボタンのタップ数（カスタムイベント）

```sql
-- アプリ側で「record_save」イベントを送信している前提
SELECT
  event_date,
  COUNT(*) AS record_count
FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260501' AND '20260531'
  AND event_name = 'record_save'
GROUP BY event_date
ORDER BY event_date
```

### 4-4. ファネル分析（記録→シェアの転換率）

```sql
WITH funnel AS (
  SELECT
    user_pseudo_id,
    COUNTIF(event_name = 'record_save') AS did_record,
    COUNTIF(event_name = 'share_card_open') AS did_share
  FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
  WHERE _TABLE_SUFFIX BETWEEN '20260501' AND '20260531'
  GROUP BY user_pseudo_id
)
SELECT
  COUNT(*) AS total_users,
  COUNTIF(did_record > 0) AS recorded_users,
  COUNTIF(did_share > 0) AS shared_users,
  ROUND(COUNTIF(did_share > 0) / NULLIF(COUNTIF(did_record > 0), 0) * 100, 1) AS share_rate_pct
FROM funnel
```

### 4-5. リテンション（翌日継続率）

```sql
WITH day0 AS (
  SELECT DISTINCT user_pseudo_id, event_date AS install_date
  FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
  WHERE event_name = 'first_open'
),
day1 AS (
  SELECT DISTINCT user_pseudo_id, event_date
  FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
)
SELECT
  d0.install_date,
  COUNT(DISTINCT d0.user_pseudo_id) AS day0_users,
  COUNT(DISTINCT d1.user_pseudo_id) AS day1_users,
  ROUND(COUNT(DISTINCT d1.user_pseudo_id) / COUNT(DISTINCT d0.user_pseudo_id) * 100, 1) AS d1_retention_pct
FROM day0 d0
LEFT JOIN day1 d1
  ON d0.user_pseudo_id = d1.user_pseudo_id
  AND PARSE_DATE('%Y%m%d', d1.event_date) = DATE_ADD(PARSE_DATE('%Y%m%d', d0.install_date), INTERVAL 1 DAY)
GROUP BY d0.install_date
ORDER BY d0.install_date
```

---

## Step 5: Flutterアプリ側の設定

GA4にイベントを送るには `firebase_analytics` パッケージを使う。

### pubspec.yaml

```yaml
dependencies:
  firebase_analytics: ^10.0.0
  firebase_core: ^2.0.0
```

### カスタムイベントの送信例（サ活ログの場合）

```dart
import 'package:firebase_analytics/firebase_analytics.dart';

final analytics = FirebaseAnalytics.instance;

// サ活記録保存時
await analytics.logEvent(
  name: 'record_save',
  parameters: {
    'sets': 3,
    'totonoi_score': 4.5,
    'facility_name': '渋谷サウナ',
  },
);

// シェアカード開いたとき
await analytics.logEvent(name: 'share_card_open');

// IAP購入完了時
await analytics.logEvent(
  name: 'purchase',
  parameters: {'price': 480},
);
```

### 自動収集されるイベント（設定不要）

| イベント名 | タイミング |
|---|---|
| `first_open` | 初回起動 |
| `session_start` | セッション開始 |
| `screen_view` | 画面遷移 |
| `app_update` | アップデート後初回起動 |

---

## Step 6: コスト管理

### 無料枠を守るクエリのコツ

```sql
-- ❌ 全期間スキャン（高コスト）
FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`

-- ✅ 日付を絞る（低コスト）
FROM `プロジェクト名.analytics_XXXXXXXXX.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260501' AND '20260531'
```

- BigQuery コンソールの右上に「このクエリは X MB を処理します」と表示されるので実行前に確認する
- 月1TB（無料枠）= 約10億行のクエリが可能。個人開発規模では超えない

---

## よくあるトラブル

| 症状 | 原因 | 対処 |
|---|---|---|
| BigQueryにテーブルが現れない | エクスポート設定直後は翌日まで待つ必要あり | 24〜48時間待つ |
| `events_*` でデータが出ない | `_TABLE_SUFFIX` の日付形式が違う | `YYYYMMDD`（ハイフンなし）で指定 |
| event_paramsが空 | アプリ側でパラメータを送っていない | Flutterの `logEvent` にparameters追加 |
| iOSシミュレータのデータが出ない | シミュレータのデータはGA4に送られない場合がある | 実機でテストする |

---

## 参考リンク

- GA4 BigQuery Export スキーマ: https://support.google.com/analytics/answer/7029846
- BigQuery コンソール: https://console.cloud.google.com/bigquery
- Firebase Analytics Flutter: https://pub.dev/packages/firebase_analytics
