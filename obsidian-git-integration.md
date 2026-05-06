# ObsidianとGitの連携ガイド

ObsidianボルトをGitで管理し、バックアップとバージョン管理を自動化する方法をまとめます。

## 基本設定

### 1. .gitignoreの設定

Obsidianの一時ファイルや個人設定を除外するための設定：

```gitignore
# Obsidianの一時ファイル
.obsidian/workspace
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache/
.obsidian/plugins/recent-files-obsidian/
.obsidian/plugins/obsidian-git/

# システムファイル
.DS_Store
Thumbs.db
*.tmp

# 個人設定（共有したくない場合）
.obsidian/hotkeys.json
.obsidian/appearance.json
```

### 2. Gitリポジトリの初期化

```bash
git init
git add .
git commit -m "Initial commit: Setup Obsidian vault"
```

## Obsidian Gitプラグインの使用

### インストール方法

1. Obsidian設定 → コミュニティプラグイン
2. 「Obsidian Git」を検索してインストール
3. プラグインを有効化

### 基本設定

```json
{
  "commitMessage": "vault backup: {{date}}",
  "autoCommitMessage": "Auto commit: {{date}}",
  "commitDateFormat": "YYYY-MM-DD HH:mm:ss",
  "autoSaveInterval": 30,
  "autoPushInterval": 0,
  "autoPullInterval": 10,
  "autoPullOnBoot": true,
  "disablePush": false,
  "pullBeforePush": true
}
```

### 自動化設定

- **自動コミット**: 30分ごと
- **自動プル**: 10分ごと
- **自動プッシュ**: 手動または設定間隔
- **起動時プル**: 有効

## Claude Codeでの自動化

### Git自動化スクリプト

```javascript
// scripts/git-automation.js
const { execSync } = require('child_process');
const fs = require('fs');

function gitCommitVault(message = '') {
  try {
    const status = execSync('git status --porcelain', { encoding: 'utf8' });
    
    if (status.trim()) {
      execSync('git add .');
      const commitMsg = message || `Vault update: ${new Date().toISOString()}`;
      execSync(`git commit -m "${commitMsg}"`);
      console.log('✅ Changes committed successfully');
      return true;
    } else {
      console.log('📝 No changes to commit');
      return false;
    }
  } catch (error) {
    console.error('❌ Git commit failed:', error.message);
    return false;
  }
}

function gitPushVault() {
  try {
    execSync('git push origin main');
    console.log('🚀 Changes pushed to remote');
    return true;
  } catch (error) {
    console.error('❌ Git push failed:', error.message);
    return false;
  }
}

module.exports = { gitCommitVault, gitPushVault };
```

### package.jsonにスクリプト追加

```json
{
  "scripts": {
    "git:commit": "node scripts/git-automation.js commit",
    "git:push": "node scripts/git-automation.js push",
    "git:backup": "node scripts/git-automation.js backup"
  }
}
```

## 推奨ワークフロー

### 毎日のルーチン

1. **起動時**: 自動プル（最新変更を取得）
2. **作業中**: 30分ごとに自動コミット
3. **終了時**: 手動プッシュまたは自動プッシュ

### 手動コマンド

```bash
# 現在の状態確認
git status

# 手動コミット
npm run git:commit

# リモートにプッシュ
npm run git:push

# 完全バックアップ
npm run git:backup
```

## マルチデバイス同期

### 設定ポイント

1. **競合の回避**: 同時編集を避ける
2. **プル頻度**: 他デバイスでの変更を定期取得
3. **プッシュタイミング**: 作業終了時に確実にプッシュ

### 競合解決

競合が発生した場合：

```bash
# マージツールを使用
git mergetool

# 手動解決後
git add .
git commit -m "Resolve merge conflicts"
```

## セキュリティ対策

### 機密情報の除外

```gitignore
# 個人情報
personal/
private/
secrets/

# APIキーなど
config/api-keys.md
*.secret
```

### リモートリポジトリの選択

- **プライベートリポジトリ**推奨
- GitHub、GitLab、Bitbucketなど
- 2要素認証の有効化

## トラブルシューティング

### よくある問題

1. **大きなファイル**: Git LFSの使用を検討
2. **頻繁な競合**: 作業タイミングの調整
3. **同期の遅延**: ネットワーク設定の確認

### リカバリ方法

```bash
# 強制リセット（注意：ローカル変更は失われる）
git reset --hard origin/main

# 特定のコミットに戻る
git revert <commit-hash>
```

## 最適化のヒント

- 画像は圧縮してからコミット
- 大きなPDFなどはGit LFSを使用
- 定期的なクリーンアップ（`git gc`）
- ブランチ戦略の検討（feature branch等）