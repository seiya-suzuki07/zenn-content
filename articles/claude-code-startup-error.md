---
title: "Claude Code エラーで起動しない？VSCode拡張の原因別トラブルシューティング完全ガイド"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "vscode", "トラブルシューティング", "ai", "開発環境"]
published: true
---

Claude Code の VSCode 拡張機能がエラーで起動しない――そんなトラブルに直面していませんか？ 2026年2月以降、特に Windows 環境で深刻な起動エラーが多発しています。本記事では、主要な原因3パターンとその具体的な解決手順をまとめました。

## Claude Code が VSCode で起動しないエラーの全体像

Claude Code の VSCode 拡張が起動しないケースは、大きく以下の3つに分類できます。

| 原因 | 影響範囲 | 緊急度 |
|------|----------|--------|
| v2.1.51 のビルドバグ | Windows 全ユーザー | 高 |
| PATH 未設定 | 全 OS | 中 |
| シンボリックリンク破損 | Windows（junction 利用者） | 中 |

エラーが発生するとエディタ上で Claude Code が一切使えなくなるため、作業が完全にストップします。以下、それぞれの原因と解決方法を解説します。

## 原因1: v2.1.51 のハードコードされた Linux パス（Windows 向けバグ）

### 症状

VSCode の拡張機能が有効化に失敗し、Developer Tools（`Ctrl+Shift+I`）のコンソールに以下のエラーが表示されます。

```text
Activating extension 'Anthropic.claude-code' failed:
TypeError: The argument 'filename' must be a file URL object,
file URL string, or absolute path string.
Received 'file:///home/runner/work/claude-cli-internal/claude-cli-internal/build-agent-sdk/sdk.mjs'
```

また、コマンドパレットで Claude 関連のコマンドを実行すると次のエラーが出ます。

```text
command 'claude-vscode.editor.openLast' not found
```

### 原因

v2.1.51 の `extension.js`（45行目付近）に、GitHub Actions CI 環境のパスがハードコードされたままリリースされてしまいました。Linux のビルドパス `/home/runner/work/...` が Windows 上では無効なため、拡張機能の初期化が失敗します。

### 解決手順

**手順1: v2.1.49 にダウングレードする**

```bash
# 現在のバージョンを確認
code --list-extensions --show-versions | grep claude

# v2.1.49 をインストール（VSIX を指定）
code --install-extension Anthropic.claude-code@2.1.49
```

**手順2: 自動更新を無効化する**

VSCode の `settings.json` に以下を追加し、自動更新でバグ版に戻されるのを防ぎます。

```json
{
  "extensions.autoUpdate": false
}
```

**手順3: 修正版がリリースされたら更新する**

GitHub Issue [#28081](https://github.com/anthropics/claude-code/issues/28081) をウォッチしておき、修正版がリリースされたら自動更新を再度有効にして更新してください。

:::message
2026年3月時点で v2.1.52 以降で修正が入っています。最新バージョンに更新できる場合はダウングレード不要です。
:::

## 原因2: PATH が通っていない（claude コマンドが見つからない）

### 症状

ターミナルで `claude` コマンドを実行すると以下のエラーが表示されます。

```bash
$ claude
zsh: command not found: claude
# または
bash: claude: command not found
```

VSCode の統合ターミナルでも同様に Claude Code が検出されません。

### 原因

`npm install -g @anthropic-ai/claude-code` でインストールした場合、npm のグローバルバイナリのディレクトリが PATH に含まれていないことがあります。また、公式インストーラーを使った場合は `~/.local/bin` にインストールされますが、これも PATH に含まれていないケースがあります。

### 解決手順

**macOS / Linux の場合:**

```bash
# npm のグローバルパスを確認
npm prefix -g
# 例: /usr/local  → バイナリは /usr/local/bin/claude

# PATH に追加（.zshrc または .bashrc）
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# インストール確認
which claude
claude --version
```

**Windows の場合:**

```powershell
# npm のグローバルパスを確認
npm prefix -g
# 例: C:\Users\username\AppData\Roaming\npm

# PowerShell でユーザー PATH に追加
$npmPath = (npm prefix -g) + "\node_modules\.bin"
[Environment]::SetEnvironmentVariable("PATH",
  "$env:PATH;$npmPath;$env:USERPROFILE\.local\bin",
  "User")
```

設定後、VSCode を完全に再起動してください（ウィンドウの再読み込みだけでは不十分な場合があります）。

:::message alert
複数の方法で Claude Code をインストールしている場合、バージョンの競合が起きることがあります。`which -a claude`（macOS/Linux）で複数のパスが出てきたら、不要な方を削除してください。
:::

## 原因3: シンボリックリンク・junction ポイントの破損

### 症状

- セッションの再開に失敗する
- 作業ディレクトリが symlink 経由でアクセスされている場合、Claude Code が正しくパスを解決できない

```text
Error: Session resume failed - working directory mismatch
Expected: /d/projects/myapp
Actual:   /c/Users/username/projects/myapp
```

### 原因

Claude Code はセッション再開時にディレクトリパスの**厳密な文字列比較**を行います。symlink や Windows の junction ポイントを経由してアクセスした場合、実体パスと symlink パスが一致せず、セッションの再開に失敗します。

### 解決手順

```bash
# symlink ではなく実体パスで作業ディレクトリを開く
# 実体パスを確認
readlink -f /path/to/symlink
# macOS の場合
realpath /path/to/symlink

# VSCode で実体パスを直接開く
code $(realpath /path/to/your/project)
```

Windows の場合は、junction ではなく実際のパスで VSCode を開いてください。

```powershell
# junction の実体パスを確認
fsutil reparsepoint query D:\projects\myapp
# 実体パスで開く
code C:\Users\username\projects\myapp
```

## 日本語ユーザー名による起動エラー（Windows 固有）

Windows でユーザー名に日本語（マルチバイト文字）が含まれている場合、Claude Code が起動に失敗するケースも報告されています。

```text
Error: ENOENT: no such file or directory, open 'C:\Users\太郎\AppData\...'
```

### 解決策

1. **英語名のローカルアカウントを新規作成**し、そちらで開発環境を構築する
2. または `CLAUDE_CONFIG_DIR` 環境変数で設定ディレクトリを ASCII パスに変更する

```powershell
# 環境変数で設定ディレクトリを変更
[Environment]::SetEnvironmentVariable("CLAUDE_CONFIG_DIR",
  "C:\claude-config", "User")
```

## それでも解決しない場合の共通チェックリスト

上記のどれにも当てはまらない場合は、以下を順番に試してください。

```bash
# 1. Node.js のバージョン確認（18以上が必要）
node --version

# 2. Claude Code を再インストール
npm uninstall -g @anthropic-ai/claude-code
npm install -g @anthropic-ai/claude-code

# 3. VSCode の拡張機能キャッシュをクリア
# macOS
rm -rf ~/Library/Application\ Support/Code/CachedExtensions
# Windows
rmdir /s %USERPROFILE%\AppData\Roaming\Code\CachedExtensions

# 4. VSCode を完全に再起動（プロセスを全て終了してから起動）

# 5. Developer Tools でエラーログを確認
# Ctrl+Shift+I (Windows) / Cmd+Shift+I (macOS)
```

## まとめ

Claude Code の VSCode 拡張が起動しないエラーは、原因を正しく特定すれば確実に解決できます。

- **v2.1.51 バグ** → ダウングレードまたは最新版へ更新
- **PATH 未設定** → シェル設定ファイルに PATH を追加
- **symlink 破損** → 実体パスでプロジェクトを開く
- **日本語ユーザー名** → 英語アカウント作成または設定ディレクトリ変更

公式の [Troubleshooting ドキュメント](https://code.claude.com/docs/en/troubleshooting) も合わせて確認してください。問題が解決しない場合は、[GitHub Issues](https://github.com/anthropics/claude-code/issues) で同様のケースがないか検索してみましょう。

---

**参考リンク:**

- [Claude Code 公式トラブルシューティング](https://code.claude.com/docs/en/troubleshooting)
- [GitHub Issue #28081: v2.1.51 hardcoded path](https://github.com/anthropics/claude-code/issues/28081)
- [GitHub Issue #28060: commands not found](https://github.com/anthropics/claude-code/issues/28060)
- [GitHub Issue #24271: symlink session resume](https://github.com/anthropics/claude-code/issues/24271)

---

:::message
AI開発の最新Tipsを毎週配信中。Bylineニュースレターに登録して、Claude Code・Cursor・GitHub Copilotなどの実践的な活用ノウハウをいち早くキャッチしましょう。
👉 [Byline ニュースレターに登録する](https://byline.dev/ai-dev-tips)
:::
