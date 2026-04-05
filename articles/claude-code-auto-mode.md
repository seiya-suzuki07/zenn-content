---
title: "Claude Code Auto Mode完全ガイド｜自動承認の設定方法と安全な使い方"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "ai", "開発効率化", "anthropic", "vscode"]
published: true
---

Claude Code Auto Modeは、2026年3月12日にリサーチプレビューとしてリリースされた新機能です。従来の`--dangerously-skip-permissions`に代わる安全な自動承認の仕組みとして注目を集めています。本記事では、Auto Modeの設定方法・従来機能との違い・安全に使うためのベストプラクティスを解説します。

## Claude Code Auto Modeとは何か

Auto Modeは、Claude Code実行中の権限プロンプト（ファイル編集やシェルコマンド実行の許可確認）をClaude自身が判断して処理する機能です。

従来、開発者は2つの選択肢しかありませんでした。

1. **毎回手動で承認する** → 作業が頻繁に中断される
2. **`--dangerously-skip-permissions`で全スキップする** → セキュリティリスクが高い

Auto Modeはこの中間に位置し、**Claudeがアクションごとにリスクを評価し、安全と判断した操作は自動承認、リスクがある操作はプロンプトを表示する**という仕組みです。

## Auto Modeの有効化方法

### CLIから有効化する

```bash
claude --enable-auto-mode
```

このフラグを付けてClaude Codeを起動すると、そのセッションでAuto Modeが有効になります。

### セッション中に切り替える

実行中のセッションで`Shift+Tab`を押すと、以下の3つのモードを切り替えられます。

| モード | 動作 |
|--------|------|
| **Default** | ファイル編集・コマンド実行の前に毎回確認 |
| **Auto-accept edits** | ファイル編集は自動承認、コマンドは確認 |
| **Plan** | 読み取り専用。計画の提示のみで変更は行わない |

### VS Code設定で初期モードを指定する

VS Codeの設定から初期の権限モードを指定できます。

```json
{
  "claudeCode.initialPermissionMode": "autoAcceptEdits"
}
```

## 権限ルールの細かな制御（settings.json）

Auto Modeの自動承認範囲は、`.claude/settings.json`で細かく制御できます。

### 許可リスト（allow）の設定

```json
{
  "permissions": {
    "allow": [
      "Edit",
      "MultiEdit",
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Bash(git status)",
      "Bash(git diff *)"
    ]
  }
}
```

`allow`に追加したツール・コマンドは、確認なしで自動実行されます。

### 拒否リスト（deny）の設定

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl *)",
      "Bash(git push --force *)",
      "WebFetch(domain:*.production.example.com)"
    ]
  }
}
```

`deny`に追加した操作は、Auto Mode中でもブロックされます。

### 権限ルールの書式

```text
Tool              → そのツールの全操作にマッチ
Tool(specifier)   → 特定パターンにマッチ
Bash(npm run *)   → npm run で始まるコマンドにマッチ
```

## Auto Mode vs --dangerously-skip-permissions 比較表

| 項目 | Auto Mode | --dangerously-skip-permissions |
|------|-----------|-------------------------------|
| **権限判断** | Claudeがリスク評価して判断 | 全操作を無条件スキップ |
| **ファイル読み取り** | 自動承認 | 自動承認 |
| **ファイル編集** | 自動承認（設定で制御可） | 自動承認 |
| **シェルコマンド** | リスク評価後に判断 | 無条件実行 |
| **外部ネットワーク** | プロンプト表示の可能性あり | 無条件実行 |
| **破壊的操作** | ブロックまたはプロンプト表示 | 無条件実行 |
| **deny設定の適用** | 適用される | 適用されない |
| **推奨環境** | 開発環境（注意付き） | サンドボックス・CI環境のみ |
| **トークン消費** | やや増加（リスク評価分） | 変化なし |
| **安全性** | 中〜高 | 低 |

## Hooksとの組み合わせ（推奨）

Claude Code v2.0以降では、**Hooks**を使った権限制御が推奨されています。Auto Modeと組み合わせることで、より堅牢な安全策を構築できます。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -q 'rm -rf'; then exit 1; fi"
      }
    ]
  }
}
```

Hooksは以下のイベントタイプをサポートしています。

- `PreToolUse` — ツール実行前に検証
- `PostToolUse` — ツール実行後に検証
- `SubagentStop` — サブエージェント停止時

`deny`ルールだけでは防ぎきれないケースがあるため、重要な制約にはHooksを併用してください。

## 安全に使うためのベストプラクティス

### 1. 段階的に権限を広げる

最初はDefault Modeで動作を確認し、信頼できるコマンドを`allow`リストに追加していくアプローチが安全です。

```json
{
  "permissions": {
    "allow": [
      "Edit",
      "Bash(npm run test *)",
      "Bash(npm run lint)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

### 2. 本番環境の認証情報を分離する

Auto Modeを使うマシンには、本番APIキーや本番DBの認証情報を置かないでください。環境変数の分離が必須です。

### 3. Gitのセーフティネットを活用する

```bash
# 作業前にブランチを切る
git checkout -b feature/auto-mode-work

# 作業後に差分を確認
git diff --stat
```

Auto Modeで自動編集されたファイルは、必ずコミット前にdiffを確認しましょう。

### 4. deny + Hooksの二重防御

`deny`ルールとHooksの両方で破壊的操作をブロックすることで、フォールバックの安全策を確保します。

### 5. プロジェクト単位で設定を管理する

`.claude/settings.json`はプロジェクトごとに設定できます。リポジトリにコミットしてチームで共有しましょう。

```bash
# プロジェクトルートに配置
.claude/
  settings.json   # プロジェクト固有の権限設定
```

## まとめ

Claude Code Auto Modeは、開発者の作業フローを大幅に効率化する一方、適切な設定なしに使うとリスクもあります。重要なポイントをまとめます。

- **`--dangerously-skip-permissions`の代替**として、リスク評価付きの自動承認を実現
- **`allow`/`deny`ルール**で自動承認の範囲を細かく制御可能
- **Hooks**との併用で二重の安全策を構築できる
- **本番認証情報の分離**と**Git差分確認**を徹底する

まだリサーチプレビュー段階のため、今後のアップデートで挙動が変わる可能性があります。公式ドキュメントを定期的にチェックしてください。

## 参考リンク

- [Configure permissions - Claude Code Docs](https://code.claude.com/docs/en/permissions)
- [Claude Code settings - Claude Code Docs](https://code.claude.com/docs/en/settings)
- [CLI reference - Claude Code Docs](https://code.claude.com/docs/en/cli-reference)

---

:::message
**AI開発の最新Tipsを毎週配信中。** Claude Code・Cursor・GitHub Copilotなど、AIコーディングツールの実践的な活用法をお届けします。
👉 [Byline ニュースレターに登録する](https://byline.dev/ai-dev-tips)
:::
