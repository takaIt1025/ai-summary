# Phase 7: 分析

```
【インプット】
- progress.json: { "phase": "published", "published_url": "https://...", "deploy_status": "deployed" }

【アウトプット】
- Googleインデックス登録完了
- スプレッドシート全タブ更新
- kpi_feedback.md 更新
- progress.json 更新: { "phase": "analyzed" }

【完了条件】
- Indexing APIへのリクエストが成功していること（またはGSC手動登録依頼済み）
```

## 実行手順

1. **デプロイ完了確認（deploy_status が "timeout" の場合のリトライ）**

   Phase 6でタイムアウトしていた場合、まずURLの疎通を確認してから進む。

   ```bash
   STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${PUBLISHED_URL}")
   if [ "${STATUS}" != "200" ]; then
     echo "URLが未応答（${STATUS}）。デプロイ完了を待ってから手動で再実行してください"
     # errors に記録してスキップ
   fi
   ```

2. **Google Indexing APIでインデックス登録**

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

3. **スプレッドシート全タブ更新**

   | タブ | 更新内容 |
   |---|---|
   | ダッシュボード | 最終更新日・総公開記事数・今月公開数・インデックス済数 |
   | 記事作成ログ | ステータス「公開済み」・URL・品質スコア記録 |
   | KW戦略 | 該当KWのStatusを「公開済」に更新 |
   | トピッククラスター | 該当KWのStatusを「公開済」・URL追加 |
   | KPIレポート | 新行追加（日付・総記事数・本日公開数・インデックス率） |

   - スプレッドシートAPI障害時: `errors` に記録して続行。後で手動同期

4. **既存記事への内部リンク自動追加**
   - 公開済み全記事のH2見出しとKWを取得
   - 新規記事のKWが自然に挿入できる箇所を特定
   - 該当する既存記事の `.md` ファイルを直接編集してリンクを追加
   - git commit / push して変更をデプロイ
   - 「内部リンク管理」タブに記録

5. **kpi_feedback.md の更新**
   - 成功パターン・失敗パターン・リライト優先度を追記
   - 翌朝のパイプラインが読み込んで品質改善に活用

6. **状態更新**
   - `progress.json` を `{ "phase": "analyzed" }` に更新
   - `errors` フィールドに残ったエラーがあればユーザーに最終レポートとして提示
