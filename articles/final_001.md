---
title: "Claude Code hooks 使い方完全ガイド【レシピ5選】"
description: "Claude Code hooksの使い方をコピペOKなレシピ形式で解説。CLAUDE.mdとの違い・exit codeの意味・よくあるエラーと解決策まで初心者向けに丁寧にまとめました。所要時間5分で最初のhookが設定できます。"
pubDate: 2026-05-12
updatedDate: 2026-05-12
slug: "claude-code-hooks-guide"
category: "Claude Code"
tags: ["Claude Code", "hooks", "AI開発", "自動化", "開発効率化"]
eyecatch: "/images/001/eyecatch.png"
faq:
  - question: "hooksはどのバージョンのClaude Codeから使えますか？"
    answer: "hooks自体は早い段階から存在しますが、ifフィールドによる細かいフィルタリングはv2.1.85以降が必要です。claude --versionで確認してください。"
  - question: "hooksのコマンドがタイムアウトする場合はどうすればいいですか？"
    answer: "デフォルトのタイムアウトは10分です。個別のhookにtimeoutフィールド（秒単位）を追加して変更できます。"
  - question: "PostToolUseでファイルを変更してもClaude Codeは検知しますか？"
    answer: "Claudeはhookによるファイル変更を直接検知しません。重要な変更の場合はClaudeに都度伝えてください。"
  - question: "hooksは並列実行されますか？"
    answer: "同一イベントに複数のhookが設定されている場合、すべて並列実行されます。副作用の順序に依存する処理は、同一のhook内で順次実行する形にしてください。"
  - question: "PermissionRequestのhookで全許可にできますか？"
    answer: "技術的にはできますが、matchers空やワイルドカードにするとすべての許可プロンプトが自動承認されます。本番環境での使用は避けてください。"
  - question: "hookのデバッグ方法を教えてください"
    answer: "claude --debug-file /tmp/claude.log で起動し、別ターミナルで tail -f /tmp/claude.log を実行してください。発火したhookの詳細（exit code・stdout・stderr）が確認できます。"
draft: false
---

# Claude Code hooks 使い方完全ガイド【レシピ5選】

CLAUDE.mdに書いた指示が守られないことはないでしょうか。「必ずprettierを実行して」と書いても、Claudeが見落とすことがあります。実はこれ、LLMの仕様通りの動作です。

<strong>hooksはその問題を根本から解決する仕組み</strong>です。ファイル編集のたびに自動フォーマット、作業完了時に通知、危険コマンドのブロック──これらをAIの判断に頼らず確実に実行できます。

この記事では、hooks初心者が今日から使えるレベルまで、設定方法・イベント一覧・コピペOKレシピを丁寧に解説します。

---

:::note
**この記事でわかること**
- Claude Code hooksとCLAUDE.mdの本質的な違い
- 5分でできる最初のhookセットアップ手順
- settings.jsonの3要素（EventName・matcher・command）の読み方
- exit code 0・2・その他の動作の違い
- すぐ使えるコピペOKレシピ5選（通知・フォーマット・ブロック・記憶再注入・自動承認）
- よくあるエラーと解決策3選
:::

---

## 1. Claude Code hooksとは──CLAUDE.mdと何が違う？

<strong>hooksはCLAUDE.mdの「お願い」と違い、毎回確実に実行されるシェルコマンド</strong>です。

CLAUDE.mdはClaudeへの指示書です。「このプロジェクトではpnpmを使ってください」と書いても、Claudeが確認を怠ることがあります。LLMの動作は確率的なので、遵守率は100%になりません。これはバグではなく、LLMの本質的な特性です。

hooksは、Claudeのライフサイクル上の特定タイミングで、シェルコマンドを確実に実行します。スクリプトとして動くため、AIの判断は一切関係ありません。

| | CLAUDE.md | hooks |
|:--|:--|:--|
| 実行される保証 | ない（確率的） | ある（確定的） |
| 用途 | プロジェクトのルール・背景知識 | 毎回必ずやりたい処理 |
| 設定場所 | CLAUDE.md ファイル | settings.json |
| 実装コスト | 低（テキスト記述のみ） | 中（JSONとシェルスクリプト） |

「毎回確実に実行してほしい処理」にはhooks、「Claudeに覚えておいてほしいルール」にはCLAUDE.md──この使い分けがhooks活用の基本方針です。

公式ドキュメント（[Claude Code Hooks ガイド](https://code.claude.com/docs/ja/hooks-guide)）には「決定論的な制御を提供する」という表現が使われており、hooksの確実性はAnthropicが明示的に保証しています。

---

## 2. 最初のhookを5分でセットアップする（通知hook）

<strong>まず通知hookを1つ設定して、hooksの動作感を体感してください。</strong>

Claudeが作業を終えてあなたの入力を待つとき、デスクトップ通知を受け取れるようにします。作業中にターミナルをずっと眺める必要がなくなります。筆者はこのhookを設定してから、他の作業と並行してClaude Codeを動かすスタイルに切り替えました。実際にターミナルを確認する頻度が半分以下になります。

### ステップ1: 設定ファイルを開く

`~/.claude/settings.json` を開きます。存在しない場合は新規作成してください。

```bash
# ファイルの場所を確認
ls ~/.claude/
```

### ステップ2: Notification hookを追加する

以下をsettings.jsonに追加します。

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

### ステップ3: /hooksで動作確認

Claude Codeのターミナルで `/hooks` と入力してください。設定済みのhook一覧が表示されます。`Notification` の横に数字が表示されていれば設定完了です。

:::tip
macOSで通知が表示されない場合は、[システム設定] → [通知] → [Script Editor] を探して [通知を許可] をオンにしてください。osascriptはScript Editor経由で通知を送るため、Script Editorの権限設定が必要です。
:::

---

## 3. settings.jsonの基本構造──EventName・matcher・command

<strong>hooksの設定は3層の構造です。この構造を理解すれば、あとはレシピを当てはめるだけです。</strong>

```json
{
  "hooks": {
    "【EventName】": [          // どのタイミングで発火するか
      {
        "matcher": "【条件】",  // さらに絞り込む条件
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

**EventName（イベント名）**は発火タイミングの指定です。`PreToolUse`（ツール実行前）、`PostToolUse`（ツール実行後）、`Stop`（Claude応答完了時）など、30種類以上が用意されています。次のセクションで初心者が使う8種類を紹介します。

**matcher（マッチャー）**はイベントをさらに絞り込むフィルターです。`PostToolUse` で `"matcher": "Edit|Write"` と書けば、ファイル編集系ツールが実行されたときだけhookが発火します。空文字 `""` にするとすべての場合で発火します。

**command（コマンド）**は実際に実行するシェルコマンドです。

複数のhookを同じイベントに登録できます。ただし、同じEventNameキーを2回書くと後の設定で上書きされます。複数のhookを追加するときは、同一キーの配列内に追記してください。

---

## 4. exit codeを理解する──0・2・その他の違い

<strong>exit codeの3パターンを覚えるだけで、ほぼすべてのhooksが実装できます。</strong>

| exit code | 意味 | 具体的な動作 |
|:--|:--|:--|
| 0 | 成功。処理を続行 | フォーマット実行・ログ記録に使う |
| 2 | ブロック。処理を中断 | 危険コマンドの拒否・ファイル保護に使う |
| その他（1など） | エラー通知して続行 | デバッグ中のエラー検知に使う |

exit 2でブロックする場合は、stderrにメッセージを書くとClaudeがその理由を理解して別のアプローチを試みます。

```bash
echo "ブロック理由をここに書く" >&2
exit 2
```

### jqでJSONを読む基本パターン

hookには、stdin経由でイベントのJSON情報が渡されます。<strong>jqを使ってその情報を取り出す</strong>のが定番パターンです。

```bash
#!/bin/bash
INPUT=$(cat)  # stdinを読む

# PreToolUse: 実行しようとしているコマンドを取得
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# PostToolUse: 編集されたファイルパスを取得
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path')
```

jqがインストールされていない場合は次のコマンドでインストールしてください。

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq
```

---

:::cta
**hooksの設定に迷ったら**  
次のセクションのコピペOKレシピをそのままsettings.jsonに貼り付けてください。変数や環境の調整は最小限で動きます。
:::

---

## 5. 主要イベント8選──どれを使えばいい？

<strong>公式には30種類以上のイベントがあります。まず次の8つだけ覚えてください。</strong>

使い分けの基準を先に示します。

- 処理を「前」に止めたい → `PreToolUse`
- 処理の「後」に何かしたい → `PostToolUse`
- Claude応答完了後に動かしたい → `Stop`
- 通知したい → `Notification`
- セッション開始・終了時 → `SessionStart` / `SessionEnd`
- プロンプト送信時 → `UserPromptSubmit`
- 特定ファイルの変更検知 → `FileChanged`

これらを一覧で確認します。

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

`PreToolUse` と `PostToolUse` はmatcherで対象ツールを絞れます。`"Edit|Write"` と指定すればファイル編集時のみ、`"Bash"` と指定すればコマンド実行時のみ発火します。大文字小文字が区別されるため注意してください（例: `edit` ではなく `Edit`）。

---

## 6. コピペOKレシピ5選──すぐ使えるhooks設定集

<strong>そのままsettings.jsonに貼り付けて使えるhooksを5つ紹介します。</strong>

複数のhooksをまとめて設定するときは、EventName単位でまとめて書いてください。同じEventNameキーを2回書くと後の設定で上書きされます。

### レシピ1: 処理完了通知

Claudeが作業を終えたらデスクトップ通知を受け取ります。macOSとLinuxのコマンドを `||` でつなぐことで、どちらの環境でも動きます。

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

Claudeがファイルを編集するたびにPrettierを自動実行。コードが常に整形された状態を保ちます。Prettierが対応していないファイル形式はスキップされるため、エラーは気にしなくて大丈夫です。

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

`rm -rf` や `DROP TABLE` を含むコマンドを<strong>exit 2でブロック</strong>します。Claudeは理由を受け取って別のアプローチを試みます。

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

Claudeのコンテキストウィンドウがいっぱいになると会話が圧縮されます。<strong>圧縮後に重要な情報が消える問題</strong>をhooksで解決できます。

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

`matcher: "compact"` を指定することで、通常のセッション開始では発火せず、圧縮後の再開時のみ動きます。毎回注入したいならCLAUDE.mdを使ってください。圧縮後のリセット対策として使うのがhooksの適切な役割分担です。

### レシピ5: 特定許可の自動承認

「Plan Modeを終了しますか？」という確認ダイアログを毎回スキップしたい場合に使います。exit codeではなく、<strong>JSON出力で `behavior: allow` を返す</strong>点が他のhooksと異なります。

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

<strong>「このhookをチームと共有するか・自分だけが使うか」で設定場所を分けてください。</strong>

| ファイルパス | スコープ | git管理 | 向いているhooks |
|:--|:--|:--|:--|
| `~/.claude/settings.json` | 全プロジェクト共通 | しない | 通知・個人ログ |
| `.claude/settings.json` | プロジェクト内共有 | する | lint・テスト・危険コマンドブロック |
| `.claude/settings.local.json` | プロジェクト内個人 | しない | 個人的なフォーマット設定 |

チームで同じClaude Codeを使うプロジェクトでは、`.claude/settings.json` をリポジトリにコミットしてください。全員が同じlintやセキュリティチェックを自動実行できます。

通知コマンドはOSによって異なります。macOSの `osascript` をLinuxメンバーのマシンで実行するとエラーになります。OSに依存する設定は `~/.claude/settings.json` か `.claude/settings.local.json` に書いてください。

---

## 8. よくあるエラーと解決策3選

<strong>hooksを設定してすぐ動く人は多くありません。よくある3つの詰まりポイントを解説します。</strong>

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
```

jqを使わずにPythonで代替することもできます。

```bash
python3 -c "import sys,json; d=json.load(sys.stdin); print(d['tool_input']['file_path'])"
```

### エラー3: Stop hookの無限ループ

Stop hookでClaudeに「タスクが完了したか確認する」処理を書くと、無限ループが発生することがあります。原因はStop hookがClaudeに追加のタスクを与え続けることです。

<strong>`stop_hook_active` フィールドを必ずチェックしてください</strong>。

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

Claude Code hooksで実現できることの整理です。

- <strong>CLAUDE.mdに書いても守られなかった処理をhooksで確実に実行</strong>できる
- exit code 0・2と簡単なbashスクリプトで、フォーマット・通知・ブロックが実装できる
- 設定場所（`~/.claude/` vs `.claude/`）の使い分けでチームと個人の設定を分離できる

まず「通知hook」から始めてください。動作を確認したら、自動フォーマット→危険コマンドブロックの順に追加していくのが定着への近道です。

hooksを使いこなしたら、MCPサーバーの設定でClaude Codeの機能をさらに拡張できます。[Claude Code MCP設定完全ガイド]（近日公開）も合わせてご確認ください。

---

**参考リンク**
- [Claude Code 公式Hooksガイド（日本語）](https://code.claude.com/docs/ja/hooks-guide)
- [Hooks リファレンス（全イベントスキーマ）](https://code.claude.com/docs/ja/hooks)
- [bash_command_validator_example.py（Anthropic公式サンプル）](https://github.com/anthropics/claude-code/blob/main/examples/hooks/bash_command_validator_example.py)

---

## FAQ

### Q1. hooksはどのバージョンのClaude Codeから使えますか？

hooks自体は早い段階から存在しますが、`if` フィールドによる細かいフィルタリングは<strong>v2.1.85以降</strong>が必要です。`claude --version` で確認してから設定してください。

### Q2. hooksのコマンドがタイムアウトする場合はどうすればいいですか？

デフォルトのタイムアウトは10分です。個別のhookに `"timeout"` フィールド（秒単位）を追加して変更できます。例: `"timeout": 30`

### Q3. PostToolUseでファイルを変更してもClaude Codeは検知しますか？

Claudeはhookによるファイル変更を直接は検知しません。重要な変更の場合はClaudeに都度伝えてください。

### Q4. hooksは並列実行されますか？

同一イベントに複数のhookが設定されている場合、すべて並列実行されます。1つのhookが `deny` を返しても、他のhookの実行は止まりません。副作用の順序に依存する処理は同一のhook内で順次実行する形にしてください。

### Q5. PermissionRequestのhookを全許可にできますか？

技術的にはできます。ただし、matcherを空またはワイルドカード（`.*`）にすると、ファイル書き込みやシェルコマンドを含む<strong>すべての許可プロンプトが自動承認されます。本番環境での使用は避けてください。</strong>

### Q6. hookのデバッグ方法を教えてください

`claude --debug-file /tmp/claude.log` でClaudeを起動し、別ターミナルで `tail -f /tmp/claude.log` を実行してください。発火したhookの詳細（exit code・stdout・stderr）が確認できます。

---

*最終更新: 2026-05-12*
