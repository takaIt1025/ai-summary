# 開発ツール・アカウント管理ガイド

作成日：2026-05-11

---

## 基本方針：個人とアプリを最初から分ける

個人アカウントとアプリ専用アカウントを分けることが最重要。

```
個人（今あるもの）          アプリ専用（新規作成）
─────────────────          ────────────────────
個人のX                    アプリのX
個人のGoogle               アプリ用Gmail
個人のGitHub               ↑ これを軸に全ツールを紐付ける
```

### 分ける理由

- ブランドとして育てやすい
- 将来アプリを売却・譲渡する際にまとめて移せる
- 個人の情報とアプリのAPIキーが混在しない

---

## アカウント管理の実務構成

### STEP1：アプリ専用メールアドレスを1つ作る

`mandala-app@gmail.com` のような専用Gmailを作り、以下を全部このメールで登録する。

```
Buffer
X（アプリアカウント）
Instagram
YouTube
Canva
Apple Developer（既存なら変更不要）
Google Play Console
AdMob
```

アプリに関わる全アカウントの入口が1箇所になる。

---

### STEP2：パスワード管理 → Bitwarden（無料）

全アカウントのID・パスワードをここに集約する。

| 項目 | 内容 |
|---|---|
| 費用 | 無料・オープンソース |
| 対応 | iPhone・Android・ブラウザ拡張 |
| 機能 | パスワード自動生成・自動入力・2FAコード管理 |
| 公式 | https://bitwarden.com |

---

### STEP3：APIキー・シークレットの管理

| 用途 | 管理場所 |
|---|---|
| GitHub Actionsで使うAPIキー | **GitHub Secrets**（無料・暗号化） |
| ローカル開発で使うキー | `.env`ファイル（`.gitignore`に追加） |
| Bitwarden | パスワードのみ（APIキーは入れない） |

**GitHub Actionsでの参照例：**

```yaml
# .github/workflows/buffer-refill.yml
env:
  BUFFER_TOKEN: ${{ secrets.BUFFER_ACCESS_TOKEN }}
```

APIキーはコードに直接書かない。GitHubのリポジトリ設定 → Secrets から登録する。

---

### STEP4：接続マップをプライベートリポジトリに記録する

GitHubのプライベートリポジトリに `accounts.md` を作り、接続関係だけメモする。
パスワードは書かず「何と何が繋がっているか」だけ記録する。

```markdown
## アカウント接続マップ

- Buffer ← X・Instagram連携済み
- GitHub Actions → Buffer API Token（Secrets登録済み）
- AdMob → Google Play Console連携済み
- Apple Developer → App Store Connect連携済み
```

---

## 全体の構造まとめ

```
アプリ専用Gmail（1つ）
    ↓
全ツールをこのメールで登録
    ↓
Bitwarden でパスワード管理
    ↓
APIキーは GitHub Secrets に集約
    ↓
接続マップを private リポジトリにメモ
```

---

## アプリが複数になった場合の横展開

同じパターンをアプリごとに繰り返す。

```
アプリA用Gmail → アプリA用ツール群
アプリB用Gmail → アプリB用ツール群
```

最初にこの構造を作っておくと、ツールが増えても管理が破綻しない。
