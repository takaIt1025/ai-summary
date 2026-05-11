# 記事構成: article_id 001

- **KW:** Claude Code hooks 使い方
- **タイトル案:** Claude Code hooks 使い方【コピペOKレシピ5選】CLAUDE.mdとの違いも解説
- **文字数目安:** 5,000〜7,000字
- **確定日:** 2026-05-12
- **ユーザー承認:** 済

---

## H2構成

### H1: Claude Code hooks 使い方【コピペOKレシピ5選】CLAUDE.mdとの違いも解説

#### ## 1. Claude Code hooksとは──CLAUDE.mdと何が違う？
- CLAUDE.mdの指示は確率的（守られないことがある）
- hooksはスクリプトなので100%確実に実行される
- 「毎回必ずやりたい処理」にはhooksが最適という結論

#### ## 2. 最初のhookを5分でセットアップする（通知hook）
- ~/.claude/settings.json の場所確認
- Notificationイベントで通知hookを追加するステップバイステップ
- /hooks コマンドで動作確認する方法

#### ## 3. settings.jsonの基本構造──EventName・matcher・command
- 3要素（EventName/matcher/command）の役割を丁寧に解説
- 実際のJSONサンプルで視覚的に理解

#### ## 4. exit codeを理解する──0・2・その他の違い
- exit 0 = 続行
- exit 2 = ブロック（PreToolUseで有効）
- その他 = エラー通知して続行
- jqでstdinからJSON取得する書き方

#### ## 5. 主要イベント8選──どれを使えばいい？
初心者に必要な8イベントに厳選:
1. Notification
2. Stop
3. PreToolUse
4. PostToolUse
5. SessionStart
6. SessionEnd
7. UserPromptSubmit
8. FileChanged

#### ## 6. コピペOKレシピ5選──すぐ使えるhooks設定集
1. 処理完了通知（macOS/Linux/Windows対応）
2. 自動フォーマット（prettier/eslint/black）
3. 危険コマンドのブロック（rm -rf等）
4. コンテキスト再注入（圧縮後の記憶リセット対策）
5. 特定許可の自動承認

#### ## 7. 設定場所の使い分け──チーム共有 vs 個人専用
- ~/.claude/settings.json → 個人全プロジェクト共通（通知・ログ）
- .claude/settings.json → プロジェクト共有（lint・テスト・セキュリティ）
- .claude/settings.local.json → 個人＋プロジェクト限定（gitignore）

#### ## 8. よくあるエラーと解決策3選
1. hookが動かない → matcherの大文字小文字問題
2. jq: command not found → インストール方法
3. Stop hookの無限ループ → stop_hook_active チェック

#### ## まとめ・次のステップ
- hooksで解決できることの再整理
- 次の記事（Claude Code MCP設定）への内部リンク
