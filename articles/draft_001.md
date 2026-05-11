# Claude Code hooks 使い方【コピペOKレシピ5選】CLAUDE.mdとの違いも解説

Claude Codeを使っていて、「CLAUDE.mdに書いた指示が守られないことがある」と感じたことはないでしょうか。実はこれ、仕様通りの動作です。CLAUDE.mdの指示はLLMへの「お願い」であり、必ず実行される保証はありません。

<strong>hooksはそれを根本的に解決する仕組み</strong>です。hooksを使えば、「ファイルを編集したら必ずprettierを走らせる」「作業が完了したら必ず通知する」といった処理を、AIの判断に依存せず確実に実行できます。

この記事では、hooks初心者が「今日から使える」レベルまで、設定方法・主要イベント・コピペOKなレシピを丁寧に解説します。

---

:::note
**この記事でわかること**
- Claude Code hooksとCLAUDE.mdの本質的な違い
- 5分でできる最初のhookセットアップ手順
- settings.jsonの3要素（EventName・matcher・command）の読み方
- exit code 0・2・その他の動作の違い
- すぐ使えるコピペOKレシピ5選
- よくあるエラーと解決策
:::

---

## 1. Claude Code hooksとは──CLAUDE.mdと何が違う？

hooksを理解するには、まずCLAUDE.mdとの違いを把握してください。

CLAUDE.mdはClaudeへの指示書です。「このプロジェクトではpnpmを使ってください」「コミット前にテストを実行してください」と書いても、Claudeがそれを見落とすことがあります。<strong>LLMの動作は確率的</strong>なので、遵守率は100%にはなりません。

hooksは、<strong>Claudeのライフサイクル上の特定のタイミングで、シェルコマンドを確実に実行する仕組み</strong>です。スクリプトとして動くため、AIの判断は一切関係ありません。

まとめると次のような使い分けになります。

| | CLAUDE.md | hooks |
|:--|:--|:--|
| 実行される保証 | ない（確率的） | ある（確定的） |
| 用途 | プロジェクトのルール・背景知識 | 毎回必ずやりたい処理 |
| 設定場所 | CLAUDE.md ファイル | settings.json |

「毎回確実に実行してほしい処理」にはhooksを使う。これがhooks活用の基本方針です。

---

## 2. 最初のhookを5分でセットアップする（通知hook）

まず通知hookを設定して、hooksがどう動くか体感してください。

Claudeが作業を終えてあなたの入力を待つとき、デスクトップ通知を受け取れるようにします。ターミナルを眺めている必要がなくなります。

### ステップ1: 設定ファイルを開く

`~/.claude/settings.json` を開きます。存在しない場合は新規作成してください。

```bash
# ファイルが存在するか確認
cat ~/.claude/settings.json
```

### ステップ2: Notification hookを追加する

以下をsettings.jsonに追加します。macOS・Linux・Windowsそれぞれのコマンドを記載します。

**macOS:**

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Codeがあなたの入力を待っています\" with title \"Claude Code\"'"
          }
        ]
      }
    ]
  }
}
```

**Linux:**

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'あなたの入力を待っています'"
          }
        ]
      }
    ]
  }
}
```

### ステップ3: 動作を確認する

Claude Codeのターミナルで `/hooks` と入力してください。設定済みのhookの一覧が表示されます。`Notification` の横に数字が表示されていれば設定完了です。

:::tip
macOSで通知が来ない場合は、システム設定 → 通知 → Script Editor を探して「通知を許可」をオンにしてください。osascriptはScript Editor経由で通知を送るため、Script Editor側の権限が必要です。
:::

---

## 3. settings.jsonの基本構造──EventName・matcher・command

hooksの設定は3層の構造になっています。一度理解すれば、あとはレシピを当てはめるだけです。

```json
{
  "hooks": {
    "【EventName】": [          // ← どのタイミングで発火するか
      {
        "matcher": "【条件】",  // ← さらに絞り込む条件
        "hooks": [
          {
            "type": "command",
            "command": "【実行するコマンド】"
          }
        ]
      }
    ]
  }
}
```

**EventName（イベント名）**は、hooksが発火するタイミングを指定します。`PreToolUse`（ツール実行前）、`PostToolUse`（ツール実行後）、`Stop`（Claude応答完了時）などがあります。次のセクションで8種類を紹介します。

**matcher（マッチャー）**は、イベントをさらに絞り込むフィルターです。`PostToolUse` で `"matcher": "Edit|Write"` と書けば、ファイル編集系のツールが実行されたときだけhookが発火します。すべての場合に発火させたいときは空文字 `""` にします。

**command（コマンド）**は、実際に実行するシェルコマンドです。複数のhookを同じイベントに登録することもできます。

---

## 4. exit codeを理解する──0・2・その他の違い

hooksはシェルコマンドの終了コード（exit code）でClaude Codeに結果を返します。<strong>この3パターンを覚えるだけで、ほぼすべてのhooksが書けます</strong>。

| exit code | 意味 | ユースケース |
|:--|:--|:--|
| 0 | 成功。処理を続行 | フォーマット・ログ記録 |
| 2 | ブロック。処理を中断 | 危険コマンドの拒否・ファイル保護 |
| その他（1等） | エラー通知して続行 | デバッグ用 |

### jqでJSONを読む基本パターン

hooksのコマンドには、stdin経由でイベントのJSON情報が渡されます。<strong>jqを使ってその情報を取り出す</strong>のが定番パターンです。

```bash
#!/bin/bash
INPUT=$(cat)  # stdinを読む

# PreToolUse の場合: 実行しようとしているコマンドを取得
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# PostToolUse の場合: 編集されたファイルパスを取得
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path')
```

jqがインストールされていない場合は次のコマンドでインストールしてください。

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq
```

exit 2でブロックする場合は、stderrにメッセージを書くとClaudeがその理由を理解して対応を変えます。

```bash
echo "ブロック理由をここに書く" >&2
exit 2
```

---

## 5. 主要イベント8選──どれを使えばいい？

公式ドキュメントには30種類以上のイベントが記載されています。ただ、日常的な開発で使うのは限られます。まず次の8つを覚えてください。

最初に使い分けの基準を示します。

- **処理を「前」に止めたい** → PreToolUse
- **処理の「後」に何かしたい** → PostToolUse
- **Claude応答完了後に動かしたい** → Stop
- **通知したい** → Notification
- **セッション開始・終了時に動かしたい** → SessionStart / SessionEnd
- **ユーザーのプロンプト送信時に動かしたい** → UserPromptSubmit
- **特定ファイルの変更を検知したい** → FileChanged

| イベント | 発火タイミング | よくある使い方 |
|:--|:--|:--|
| `Notification` | Claudeが入力を待つとき | デスクトップ通知 |
| `Stop` | Claude応答完了時 | 通知・後処理 |
| `PreToolUse` | ツール実行前 | 危険コマンドのブロック |
| `PostToolUse` | ツール実行後 | 自動フォーマット |
| `SessionStart` | セッション開始時 | コンテキスト注入・環境設定 |
| `SessionEnd` | セッション終了時 | クリーンアップ |
| `UserPromptSubmit` | プロンプト送信前 | 内容のログ記録 |
| `FileChanged` | 監視ファイルが変更時 | 環境変数の再読み込み |

`PreToolUse` と `PostToolUse` は `matcher` で対象ツールを絞れます。`"Edit|Write"` と指定すればファイル編集時のみ、`"Bash"` と指定すればコマンド実行時のみ発火します。

---

## 6. コピペOKレシピ5選──すぐ使えるhooks設定集

実際に使えるhooksを5つ紹介します。settings.jsonに追記するだけで動きます。

複数のhooksをまとめて設定するときは、EventName単位でまとめて書いてください。同じEventNameを2回書くと後の設定が上書きされます。

### レシピ1: 処理完了通知

Claudeが作業を終えたらデスクトップ通知を受け取ります。MacとLinuxのコマンドを `||` でつなぐことで、どちらの環境でも動くようにしています。

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"作業完了\" with title \"Claude Code\"' 2>/dev/null || notify-send 'Claude Code' '作業完了'"
          }
        ]
      }
    ]
  }
}
```

### レシピ2: 自動フォーマット

Claudeがファイルを編集するたびにPrettierを自動実行します。コードが常に整形された状態を保ちます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

Pythonプロジェクトにはblackを使ってください。

```bash
jq -r '.tool_input.file_path' | xargs black 2>/dev/null || true
```

### レシピ3: 危険コマンドのブロック

`rm -rf` や `DROP TABLE` を含むコマンドをClaude Codeが実行しようとした場合、<strong>exit 2でブロック</strong>します。Claudeは理由を受け取って別のアプローチを試みます。

`.claude/hooks/block-dangerous.sh` として保存してください。

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

DANGEROUS_PATTERNS=("rm -rf" "DROP TABLE" "DROP DATABASE" "format c:")

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qi "$pattern"; then
    echo "ブロック: 危険なコマンドを検出しました: $pattern" >&2
    exit 2
  fi
done

exit 0
```

```bash
chmod +x .claude/hooks/block-dangerous.sh
```

settings.jsonに次を追加します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous.sh"
          }
        ]
      }
    ]
  }
}
```

### レシピ4: コンテキスト再注入（圧縮後の記憶リセット対策）

Claudeのコンテキストウィンドウがいっぱいになると、会話が圧縮されます。<strong>圧縮後に重要な情報が消えてしまう問題</strong>をhooksで解決できます。

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'プロジェクトのルール再確認: npmではなくpnpmを使うこと。コミット前にpnpm testを実行すること。'"
          }
        ]
      }
    ]
  }
}
```

`matcher: "compact"` を指定することで、通常のセッション開始では発火せず、圧縮後の再開時のみ動きます。

### レシピ5: 特定許可の自動承認

「Plan Modeを終了しますか？」という確認ダイアログを毎回スキップしたい場合に使います。exit codeではなく、<strong>JSON出力で `behavior: allow` を返す</strong>点がポイントです。

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\": {\"hookEventName\": \"PermissionRequest\", \"decision\": {\"behavior\": \"allow\"}}}'"
          }
        ]
      }
    ]
  }
}
```

---

## 7. 設定場所の使い分け──チーム共有 vs 個人専用

hooksの設定場所は3つあります。<strong>「このhookをチームと共有するか・自分だけが使うか」で設定場所を分けてください</strong>。

| ファイルパス | スコープ | git管理 | 向いているhooks |
|:--|:--|:--|:--|
| `~/.claude/settings.json` | 全プロジェクト共通 | しない | 通知・個人ログ |
| `.claude/settings.json` | プロジェクト内共有 | する | lint・テスト・危険コマンドブロック |
| `.claude/settings.local.json` | プロジェクト内個人 | しない（gitignore推奨） | 個人的なフォーマット設定 |

チームで同じClaude Codeを使うプロジェクトでは、`.claude/settings.json` をリポジトリにコミットしてください。全員が同じlintやセキュリティチェックを自動実行できます。

通知コマンドはOSによって異なります。macOSの `osascript` をLinuxメンバーのマシンで実行するとエラーになります。そういった個人依存の設定は `~/.claude/settings.json` か `.claude/settings.local.json` に書いてください。

---

## 8. よくあるエラーと解決策3選

hooksを設定してすぐ動く人は多くありません。よくある3つの詰まりポイントと解決策を示します。

### エラー1: hookが動かない（matcherの大文字小文字問題）

matcherは<strong>大文字小文字を区別します</strong>。ツール名を小文字で書くと一致しません。

```json
// NG: 動かない
"matcher": "edit|write"

// OK: 正しい
"matcher": "Edit|Write"
```

確認方法は `/hooks` コマンドです。設定済みのhookが表示されないときは、settings.jsonのJSONが正しいかも確認してください。末尾のカンマやコメント（`//`）はJSONでは使えません。

### エラー2: jq: command not found

jqがインストールされていないとhookが動きません。エラーログに `jq: command not found` と表示されます。

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq

# jqなしで動かしたい場合（Python代替）
python3 -c "import sys,json; d=json.load(sys.stdin); print(d['tool_input']['file_path'])"
```

### エラー3: Stop hookの無限ループ

Stop hookでClaudeに「タスクが完了したか確認する」処理を書くと、無限ループが発生することがあります。

原因は、Stop hookがClaudeに追加のタスクを与え続けることです。<strong>`stop_hook_active` フィールドを必ずチェックしてください</strong>。

```bash
#!/bin/bash
INPUT=$(cat)

# すでにStop hookが発動中なら即終了
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0
fi

# ここに実際の処理を書く
```

---

## まとめ・次のステップ

Claude Code hooksで実現できることを整理します。

- <strong>CLAUDE.mdに書いても守られなかった処理</strong>をhooksで確実に実行できる
- exit code 0・2と簡単なbashスクリプトで、フォーマット・通知・ブロックが実装できる
- 設定場所（`~/.claude/` vs `.claude/`）の使い分けでチームと個人の設定を分離できる

まず「通知hook」から始めてください。動作を確認したら、自動フォーマットや危険コマンドブロックと順番に追加していくのが定着への近道です。

hooksを使いこなしたら、次はMCPサーバーの設定でClaude Codeの機能をさらに拡張できます。[Claude Code MCP設定ガイド]（準備中）も合わせて参照してください。

---

## FAQ

### Q1. hooksはどのバージョンのClaude Codeから使えますか？

hooks自体は早い段階から存在しますが、`if` フィールドによる細かいフィルタリングはv2.1.85以降が必要です。`claude --version` で確認してから設定してください。

### Q2. hooksのコマンドがタイムアウトする場合はどうすればいいですか？

デフォルトのタイムアウトは10分です。個別のhookに `"timeout"` フィールド（秒単位）を追加することで変更できます。例：`"timeout": 30`

### Q3. PostToolUseでファイルを変更してもClaude Codeは検知しますか？

Claudeはhookによるファイル変更を直接は検知しません。フォーマッターが変更したファイルを次のターンでClaudeが読み直すかどうかは、タスクの内容次第です。重要な変更の場合はClaudeにその旨を伝えてください。

### Q4. hooksは並列実行されますか？

同一イベントに複数のhookが設定されている場合、すべて並列実行されます。一方のhookが `deny` を返しても、他のhookの実行は止まりません。副作用の順序に依存する処理は、同一のhook内でスクリプトを順次実行する形にしてください。

### Q5. PermissionRequestのhookをワイルドカードで全許可にできますか？

技術的にはできます。ただし、matcher を空にするかワイルドカード（`.*`）にすると、ファイル書き込みやシェルコマンドを含むすべての許可プロンプトが自動承認されます。<strong>本番環境での使用は避けてください</strong>。

### Q6. hookのデバッグ方法を教えてください

`claude --debug-file /tmp/claude.log` でClaudeを起動し、別ターミナルで `tail -f /tmp/claude.log` を実行してください。発火したhookの詳細（exit code・stdout・stderr）が確認できます。

---

*最終更新: 2026-05-12*
