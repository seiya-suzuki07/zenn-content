---
title: "【MCP設定エラー】「Could not attach to MCP server」の原因と解決法まとめ"
emoji: "🔧"
type: "tech"
topics: ["MCP", "ClaudeDesktop", "AI", "トラブルシューティング", "環境構築"]
published: true
---

MCP（Model Context Protocol）サーバーの設定でエラーが出て接続できない――Claude Desktopで「Could not attach to MCP server」というエラーに遭遇した方は多いのではないでしょうか。本記事では、MCPサーバー接続エラーの3大原因と、それぞれの具体的な解決策をコード例付きで解説します。

## MCPサーバー接続エラーとは

Claude DesktopでMCPサーバーを設定した直後、ツールアイコンが赤くなり、以下のようなエラーが表示されるケースが頻発しています。

```text
Could not attach to MCP server {サーバー名}
```

あるいはログには次のようなエラーが記録されます。

```text
Error: spawn npx ENOENT
Error: spawn uv ENOENT
```

この `ENOENT`（Error NO ENTry）は、指定されたコマンドがPATH上に見つからないことを意味します。つまり、Claude Desktopがサーバー起動コマンドを実行しようとしたが、**コマンド自体が見つからなかった**のです。

## なぜClaude Desktopだけで起きるのか？

ターミナルでは `npx` も `uv` も問題なく動くのに、Claude Desktopでは失敗する。この現象の根本原因は、**macOSのGUIアプリは `~/.zshrc` を読まない**という仕様にあります。

macOSでは、Finderやランチパッドから起動したアプリ（= GUIアプリ）はログインシェルの設定ファイル（`~/.zshrc`）を読み込みません。つまり：

- NVMが `~/.zshrc` で設定したPATH → **GUIアプリには見えない**
- Homebrew経由でインストールした `uv` のPATH → **GUIアプリには見えない**
- `~/.zshrc` で `export PATH` した内容 → **GUIアプリには見えない**

Claude Desktopが子プロセスとしてMCPサーバーを起動する際、親（Claude Desktop）が持つ限定的なPATHしか継承されないため、`npx` や `uv` が見つからないのです。

## 3大原因と解決策

### 原因1: npxのパスが解決できない（NVM環境）

NVM（Node Version Manager）を使っている場合、`npx` は `~/.nvm/versions/node/vXX.X.X/bin/npx` にインストールされています。このパスは `~/.zshrc` のNVM初期化スクリプトで設定されるため、Claude Desktopからは見えません。

**解決策A: 絶対パスを指定する**

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "/Users/username/.nvm/versions/node/v22.12.0/bin/npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/Desktop"
      ]
    }
  }
}
```

自分の環境のパスは以下で確認できます。

```bash
which npx
# 例: /Users/username/.nvm/versions/node/v22.12.0/bin/npx
```

**解決策B: グローバルインストール + nodeの絶対パスを使う**

`npx` を経由せず、パッケージをグローバルにインストールして `node` で直接実行する方法です。

```bash
npm install -g @modelcontextprotocol/server-filesystem
```

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "/Users/username/.nvm/versions/node/v22.12.0/bin/node",
      "args": [
        "/Users/username/.nvm/versions/node/v22.12.0/lib/node_modules/@modelcontextprotocol/server-filesystem/dist/index.js",
        "/Users/username/Desktop"
      ]
    }
  }
}
```

### 原因2: NVM環境変数が子プロセスに継承されない

NVM自体の環境変数（`NVM_DIR` など）が未設定のため、`nvm` コマンド経由の解決が丸ごと失敗するパターンです。

**解決策: env フィールドでPATHを明示する**

`claude_desktop_config.json` の各サーバー設定に `env` フィールドを追加し、必要なPATHを直接渡します。

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/username/Desktop"],
      "env": {
        "PATH": "/Users/username/.nvm/versions/node/v22.12.0/bin:/usr/local/bin:/usr/bin:/bin"
      }
    }
  }
}
```

この方法なら `command` を絶対パスにしなくても、`env.PATH` に含まれるディレクトリから `npx` を探してくれます。

### 原因3: uvコマンドが見つからない（Pythonサーバー）

Pythonベースのサーバー（`uvx` 経由で起動するもの）では、`uv` のパスが見えずに失敗します。

```text
Error: spawn uv ENOENT
```

**解決策A: 絶対パスを使う**

```bash
which uv
# 例: /Users/username/.local/bin/uv
```

```json
{
  "mcpServers": {
    "mcp-server-fetch": {
      "command": "/Users/username/.local/bin/uvx",
      "args": ["mcp-server-fetch"]
    }
  }
}
```

**解決策B: /usr/local/bin にシンボリックリンクを作成する**

macOSのGUIアプリは `/usr/local/bin` をデフォルトでPATHに含んでいるため、ここにシンボリックリンクを置くと解決します。

```bash
sudo ln -sf $(which uv) /usr/local/bin/uv
sudo ln -sf $(which uvx) /usr/local/bin/uvx
```

この方法は `uv` をアップデートしても自動的に追従するのがメリットです。

**解決策C: ~/.zprofile に PATH を追加する**

`~/.zshrc` ではなく `~/.zprofile` にPATH設定を書くと、ログイン時に一度だけ実行されるため、一部のGUIアプリでも認識される場合があります。

```bash
# ~/.zprofile に追加
export PATH="$HOME/.local/bin:$PATH"
```

設定後はmacOSにログインし直す必要があります。

## デバッグの手順

上記の対策を試す前に、まずログを確認しましょう。

```bash
# macOS: Claude Desktopのログを確認
tail -f ~/Library/Logs/Claude/mcp*.log
```

ログに表示されるエラーメッセージから、どの原因に該当するか特定できます。

| ログのキーワード | 原因 |
|---|---|
| `spawn npx ENOENT` | npxパス問題（原因1） |
| `spawn node ENOENT` | Node.jsパス問題（原因1・2） |
| `spawn uv ENOENT` / `spawn uvx ENOENT` | uvパス問題（原因3） |
| `TIMEOUT` / `Server failed to start` | 初回ビルドのタイムアウト（再起動で解決することが多い） |

## まとめ: チェックリスト

MCPサーバー接続エラーが出たら、以下を順にチェックしてください。

1. **ログを確認する** — `~/Library/Logs/Claude/mcp*.log` でエラー内容を特定
2. **`which` でパスを確認する** — `which npx`、`which uv` で絶対パスを取得
3. **設定ファイルを絶対パスに書き換える** — `command` フィールドに絶対パスを指定
4. **envフィールドでPATHを渡す** — 必要に応じて `env.PATH` を設定
5. **Claude Desktopを再起動する** — 設定変更後は必ず再起動

「ターミナルでは動くのにClaude Desktopでは動かない」の正体は、GUIアプリのPATH継承問題です。この仕組みを理解しておけば、今後新しいMCPサーバーを追加する際にもハマることはなくなるでしょう。

---

:::message
**AI開発の最新Tipsを毎週配信中** -- MCPの活用法、Claude Codeの実践テクニック、AIエージェント開発のベストプラクティスなど、現場で使える情報をニュースレター「Byline」でお届けしています。
👉 [Bylineに登録する](https://byline.dev/ai-dev-tips)
:::
