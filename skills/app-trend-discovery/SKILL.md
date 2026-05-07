---
description: App Storeトップチャート・Xトレンド・Google Suggestを横断分析し、今ヒットしそうなアプリアイデアをTOP5で抽出する
---

# アプリトレンド発見スキル

完全無料のデータソースからリアルタイムトレンドを収集し、  
「今作るとヒットしそうなアプリアイデア」を個人開発視点でTOP5抽出します。

## プロンプト

```
あなたは個人開発アプリのトレンドリサーチ専門家です。
以下の手順でリアルタイムデータを収集・分析し、今ヒットしそうなアプリアイデアをTOP5で提示してください。

【調査方針】
- 対象: iOS（App Store）日本市場
- 開発者: 個人開発者（1人）
- 使用ツール: 完全無料のAPI/エンドポイントのみ
- 出力言語: 日本語
- ジャンル絞り込み: $ARGUMENTS（省略時は全ジャンル）
  ※ ジャンルが渡された場合は、そのジャンルに関連するトレンドを重点的に調査する

---

【作業手順】

### Step 1: App Store トップチャートを収集する

iTunes RSS Feed（公式・無料）で日本のトップチャートを全カテゴリ取得する。

```bash
# 全体トップ100（無料アプリ）
curl -s "https://itunes.apple.com/jp/rss/topfreeapplications/limit=100/json" | python3 -c "
import sys, json
from collections import Counter
data = json.load(sys.stdin)
entries = data['feed']['entry']
# カテゴリ別集計
cats = Counter(e['category']['attributes']['label'] for e in entries)
print('=== カテゴリ分布（上位10）===')
for cat, count in cats.most_common(10):
    print(f'  {cat}: {count}件')
print()
print('=== TOP20アプリ ===')
for i, e in enumerate(entries[:20], 1):
    print(f'{i:2}. {e[\"im:name\"][\"label\"]} | {e[\"category\"][\"attributes\"][\"label\"]}')
"
```

カテゴリ別トップも取得する（主要カテゴリのID）:
| カテゴリ | genre ID |
|---|---|
| ゲーム | 6014 |
| ビジネス | 6000 |
| 仕事効率化 | 6007 |
| 健康/フィットネス | 6013 |
| 教育 | 6017 |
| ライフスタイル | 6012 |
| ファイナンス | 6015 |
| ソーシャルネットワーキング | 6005 |
| フード/ドリンク | 6023 |
| 旅行 | 6003 |

```bash
for genre_id in 6014 6000 6007 6013 6017 6012 6015; do
  curl -s "https://itunes.apple.com/jp/rss/topfreeapplications/limit=10/genre=${genre_id}/json" | python3 -c "
import sys, json
data = json.load(sys.stdin)
title = data['feed']['title']['label']
apps = [e['im:name']['label'] for e in data['feed'].get('entry', [])[:5]]
print(f'{title}: {apps}')
"
  sleep 1
done
```

また、新着アプリもチェックして「最近登場した注目アプリ」を把握する:
```bash
curl -s "https://itunes.apple.com/jp/rss/newapplications/limit=20/json" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for e in data['feed']['entry'][:10]:
    print(e['im:name']['label'], '|', e['category']['attributes']['label'])
"
```

**分析ポイント:**
- 特定カテゴリへの集中（例: AIアプリが仕事効率化に偏在）
- 新着アプリの中でカテゴリのパターン
- 有名アプリが少ないカテゴリ（個人開発の隙間）

---

### Step 2: X（Twitter）トレンドを収集する

trends24.in をスクレイピングして、日本のXリアルタイムトレンドを取得する。

```bash
curl -s "https://trends24.in/japan/" -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" | python3 -c "
import sys, re
html = sys.stdin.read()
# trend-card内のリンクテキストを抽出
trends = re.findall(r'<a[^>]*>([^<]{2,40})</a>', html)
# ハッシュタグや短すぎるもの以外を抽出
filtered = [t.strip() for t in trends if len(t.strip()) >= 3 and not t.startswith('http')]
# 重複除去
seen = set()
unique = []
for t in filtered:
    if t not in seen and not t.startswith(('Japan', 'Trend', '20', 'http', 'Top')):
        seen.add(t)
        unique.append(t)
print('=== Xトレンド（日本）===')
for i, t in enumerate(unique[:30], 1):
    print(f'{i:2}. {t}')
"
```

**分析ポイント:**
- ニュース・時事系は除外（アプリ化が難しい）
- エンタメ・趣味・生活習慣系に注目
- 繰り返し登場するテーマ（一過性ではない需要）

---

### Step 3: Google Suggest でトレンドワードの需要を確認する

Step 1・2で発見したトレンドワードについて、アプリ需要があるか Suggest で確認する。

```bash
python3 -c "
from urllib.parse import quote
import subprocess, json

# Step 1-2で見つけたトレンドワード（5〜10個）を確認
trend_words = ['ここにStep1-2で見つけたワードを入れる']

for word in trend_words:
    url = f'https://suggestqueries.google.com/complete/search?client=firefox&hl=ja&q={quote(word + \"アプリ\")}'
    result = subprocess.run(['curl', '-s', url], capture_output=True)
    raw = result.stdout
    for enc in ['utf-8', 'cp932', 'shift_jis']:
        try:
            data = json.loads(raw.decode(enc))
            suggests = data[1][:5]
            print(f'{word}: {suggests}')
            break
        except:
            continue
"
```

---

### Step 4: 競合の薄さをチェックする

Suggest でアプリ需要が確認できたワードについて、iTunes Search APIで競合密度を確認する。

```bash
python3 -c "
from urllib.parse import quote
import subprocess, json

words = ['ここにStep3で需要確認できたワードを入れる']
for word in words:
    url = f'https://itunes.apple.com/search?term={quote(word)}&country=jp&entity=software&limit=10&lang=ja_jp'
    result = subprocess.run(['curl', '-s', url], capture_output=True)
    data = json.loads(result.stdout.decode('utf-8'))
    count = data['resultCount']
    apps = data['results'][:3]
    max_reviews = max((a.get('userRatingCount', 0) for a in apps), default=0)
    avg_rating = sum(a.get('averageUserRating', 0) for a in apps) / max(len(apps), 1)
    print(f'{word}: {count}件 | 最大レビュー{max_reviews:,}件 | 平均★{avg_rating:.1f}')
"
```

---

### Step 5: トレンド × 競合マップを作る

収集データを整理して、以下のマトリクスに配置する:

```
           競合が薄い          競合が多い
           ┌──────────────────────────────┐
トレンド   │  🎯 狙い目ゾーン  │  🔥 激戦区  │
上昇中     │  （今すぐ作れ）   │ （差別化必須）│
           ├──────────────────┼────────────│
トレンド   │  💤 待ち or        │  ⛔ 見送り  │
低迷中     │  先行者利益ゾーン  │             │
           └──────────────────────────────┘
```

**🎯 狙い目ゾーンの条件:**
- Xトレンド or App Storeカテゴリ急増で「今注目されている」
- App Store検索ヒット数が20件以下 or 上位のレビュー数が少ない（< 1,000件）
- Google Suggestに「アプリ」「無料」が出る

---

### Step 6: アイデアを具体化して簡易スコアリングする

狙い目ゾーンのテーマを「アプリアイデア」に変換し、3軸で素早く採点する（詳細はapp-hit-predictionスキルに委ねる）。

| アイデア | 需要(1-5) | 競合隙間(1-5) | 個人実現性(1-5) | 合計 |
|---|---|---|---|---|

採点基準（簡易版）:
- **需要**: Suggestに「アプリ」が出れば+2、「無料」「おすすめ」も出れば+1ずつ
- **競合隙間**: ヒット < 10件なら5点、10〜20件なら4点、21〜50件なら3点、50件超なら2点以下
- **個人実現性**: ローカル完結なら5点、軽いAPI連携で4点、バックエンド必須で3点以下

---

### Step 7: アイデアをTOP5にランキングして提示する

Step 6のスコア順に並べ、各アイデアについて以下を出力する:
- **なぜ今ヒットするか**（トレンドとの接点）
- **MVP（最小機能）の定義**（1〜2機能に絞る）
- **想定ユーザー**と**マネタイズ仮説**
- **リスク・懸念点**

---

### Step 8: レポートを保存する

`trend-discovery/[YYYYMMDD_HHMMSS]_trend.md` 形式で保存する。
trend-discovery ディレクトリが存在しない場合は作成する。

---

【回答形式】

# アプリトレンド発見レポート

調査日時: [現在の日時]
対象: iOS / 日本 / 個人開発視点
ジャンル絞り込み: [指定があれば / なければ「全ジャンル」]

---

## 今日のトレンドサマリー

### App Store 注目カテゴリ（上位3）
| カテゴリ | 傾向 | 個人開発の余地 |
|---|---|---|

### Xトレンド アプリ化候補（上位5）
| トレンドワード | 推定需要 | 一言コメント |
|---|---|---|

---

## 🎯 狙い目アイデア TOP5

### 1位: [アイデア名]
- **トレンドとの接点**: ...（どのデータソースで発見したか）
- **MVP機能**: ...（最小限これだけあれば出せる）
- **想定ユーザー**: ...
- **マネタイズ仮説**: ...（無料+IAP / サブスク / 買い切り）
- **簡易スコア**: 需要X + 競合X + 実現性X = **X/15点**
- **リスク**: ...

### 2位〜5位
（同形式で続く）

---

## トレンド × 競合マップ

```
           競合が薄い          競合が多い
           ┌──────────────────────────────┐
トレンド   │  [アイデアA]       │  [アイデアD] │
上昇中     │  [アイデアB]       │  [アイデアE] │
           ├──────────────────┼────────────│
トレンド   │  [アイデアC]       │              │
低迷中     │                    │              │
           └──────────────────────────────┘
```

---

## 今週の「見送り」判定

以下はトレンドがあっても個人開発では難しいと判断したもの:
- [テーマ]: 理由（例: 競合が強すぎる / 実現難度が高い）

---

## 次のアクション

- [ ] 1位アイデアを `/app-hit-prediction` で詳細評価する
- [ ] MVPのスコープを決めてプロトタイプ着手

---

## 付録: 収集データ

### App Store トップ20（取得時点）
[実際のデータ]

### Xトレンド（取得時点）
[実際のデータ]

- 調査ツール: iTunes RSS Feed / trends24.in / Google Suggest
- 調査日: [日付]
```

---

## 使用例

### 例1: 全ジャンル横断でトレンドを探す
```
/app-trend-discovery
```
→ 全カテゴリのApp StoreチャートとXトレンドを横断してTOP5を抽出

### 例2: ジャンルを絞って探す
```
/app-trend-discovery 健康・フィットネス
```
→ 健康カテゴリのチャートとそれに関連するXトレンドに絞って分析

### 例3: 複数ジャンルで比較
```
/app-trend-discovery 仕事効率化 ライフスタイル
```
→ 指定ジャンル横断でトレンドを比較してアイデアを抽出

---

## 他スキルとの連携フロー

```
/app-trend-discovery          ← このスキル（今週のアイデアを発見）
        ↓ TOP5アイデアを渡す
/app-hit-prediction [アイデア] ← 詳細なGo/No-Go判定
        ↓ Goになったアイデアを渡す
/app-keyword-research [KW]    ← ASO最適化
```

---

## 技術的制約と注意

- **Google Play**: 無料の公式APIなし。App Storeのチャートで代替し、Google Play側は手動確認を推奨
- **Xトレンド**: trends24.inはX非公式のアグリゲーター。サイト構造変更でスクレイピングが壊れる可能性があるため、取得失敗時はスキップして続行する
- **pytrends（Google Trends）**: 要インストール（`pip install pytrends`）。未インストール時はGoogle SuggestとApp Storeで代替
- **iTunes RSS レート制限**: カテゴリ別ループ時は `sleep 1` を入れること
- 本スキルで出るアイデアは「トレンド起点」。深いユーザー課題の検証には `/problem-discovery-business-creation` スキルを補完利用すること
