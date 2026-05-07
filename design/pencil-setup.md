# Pencil（Evolus Pencil）セットアップガイド

バージョン: v3.1.1  
対応OS: macOS（Intel / Apple Silicon 両対応）  
料金: 無料・オープンソース

---

## インストール手順

### 1. dmgファイルをダウンロード

公式サイトのダウンロードページから **macOS Universal** を選択してダウンロードします。

```
https://pencil.evolus.vn/Downloads.html
```

### 2. インストール

1. ダウンロードした `.dmg` ファイルを開く
2. `Pencil.app` を `アプリケーション` フォルダにドラッグ
3. `.dmg` をアンマウント（取り出し）

### 3. 初回起動時の注意（macOS Gatekeeper）

Apple公式以外のアプリのため、そのままダブルクリックすると「開発元を確認できない」と表示される場合があります。

**対処方法:**

```bash
# ターミナルで以下を実行してから起動する
xattr -dr com.apple.quarantine /Applications/Pencil.app
```

または:

1. `Finder` でアプリを右クリック
2. `開く` を選択
3. 確認ダイアログで `開く` をクリック

---

## 基本的な使い方

### 画面（Page）の作成

- 左パネルの `+` からページを追加
- 1画面 = 1ページとして管理するのが基本

### iOS向けステンシルの使い方

Pencil には iOS の UI 部品（ステンシル）が組み込まれています。

1. 右パネルの `Stencil Collections` を開く
2. `iOS Mockup` を選択
3. ボタン・テキストフィールド・ナビゲーションバー等をドラッグ＆ドロップ

### 画面遷移のリンク設定

1. 遷移元の要素を選択
2. 右クリック → `Link to` → 遷移先のページを指定
3. HTMLエクスポートするとクリッカブルなプロトタイプになる

### エクスポート

- `File` → `Export Document` → PNG / PDF / HTML から選択
- **PNG**: 各画面を画像として書き出し（開発者との共有に最適）
- **HTML**: クリッカブルなプロトタイプとして書き出し

---

## サ活ログ MVPで作成する画面

[画面設計書](./sakatsu-log-screens.md) を参照しながら以下9画面を作成します。

| 優先度 | 画面 | ファイル名（ページ名） |
|---|---|---|
| 🔴 高 | S3 ホーム（記録一覧） | 03_home |
| 🔴 高 | S4 記録入力（ボトムシート） | 04_record_input |
| 🔴 高 | S6 シェアカード | 06_share_card |
| 🔴 高 | S7 カレンダー | 07_calendar |
| 🟡 中 | S5 記録詳細 | 05_record_detail |
| 🟡 中 | S8 設定 | 08_settings |
| 🟢 低 | S1 スプラッシュ | 01_splash |
| 🟢 低 | S2 オンボーディング | 02_onboarding |
| 🟢 低 | S9 Pro購入 | 09_pro_purchase |

---

## 関連リンク

- 公式サイト: https://pencil.evolus.vn/
- GitHub: https://github.com/evolus/pencil
