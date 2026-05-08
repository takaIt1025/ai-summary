# Phase 7: 分析

```
【インプット】
- progress.json: { "phase": "published", "wp_post_id": 1234, "published_url": "https://..." }

【アウトプット】
- Googleインデックス登録完了
- スプレッドシート全タブ更新
- kpi_feedback.md 更新
- progress.json 更新: { "phase": "analyzed" }

【完了条件】
- Indexing APIへのリクエストが成功していること（またはGSC手動登録依頼済み）
```

## 実行手順

1. **Google Indexing APIでインデックス登録**
   ```python
   import json
   from google.oauth2 import service_account
   from googleapiclient.discovery import build

   with open('automation/logs/progress.json') as f:
       progress = json.load(f)
   published_url = progress['published_url']  # progress.json から取得

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

2. **スプレッドシート全タブ更新**

   | タブ | 更新内容 |
   |---|---|
   | ダッシュボード | 最終更新日・総公開記事数・今月公開数・インデックス済数 |
   | 記事作成ログ | ステータス「公開済み」・URL・品質スコア記録 |
   | KW戦略 | 該当KWのStatusを「公開済」に更新 |
   | トピッククラスター | 該当KWのStatusを「公開済」・URL追加 |
   | KPIレポート | 新行追加（日付・総記事数・本日公開数・インデックス率） |

   - スプレッドシートAPI障害時: `errors` に記録して続行。後で手動同期

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
