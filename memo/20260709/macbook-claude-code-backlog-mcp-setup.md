# MacBook で Claude Code をインストールして Backlog MCP を使うまで

## この手順の前提

対象は macOS の MacBook。

この手順では、Claude Code をインストールし、Claude Code から Backlog MCP を使って Backlog の課題を読めるところまでを扱う。

Backlog MCP サーバーの具体的なパッケージ名や起動コマンドは、社内で採用する MCP サーバーの README に合わせて差し替える。

## 事前に用意するもの

- Claude Code を利用できる Claude アカウント
- MacBook
- ターミナル
- Backlog のスペース URL
- Backlog API キー
- Backlog MCP サーバーの導入コマンド

Backlog のスペース URL 例。

```text
https://example.backlog.com
```

Backlog API キーは秘密情報なので、チャットや資料に貼らない。

## 1. Claude Code をインストールする

公式手順では、macOS は Native Install が推奨されている。

ターミナルで以下を実行する。

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Homebrew で入れる場合は以下。

```bash
brew install --cask claude-code
```

ただし Homebrew 版は自動更新されないため、通常は Native Install を使うのが分かりやすい。

## 2. インストール確認

```bash
claude --version
```

より詳しく確認する場合。

```bash
claude doctor
```

`command not found: claude` になる場合は、ターミナルを開き直す。

それでも解決しない場合は、インストール先が `PATH` に入っていない可能性がある。

## 3. Claude Code にログインする

作業したいリポジトリに移動して、Claude Code を起動する。

```bash
cd /path/to/your/project
claude
```

初回起動時にブラウザでログインを求められるので、案内に従ってログインする。

起動後、まずは読み取りだけの依頼で確認する。

```text
このリポジトリの構成を読んで、何のためのリポジトリか説明して
```

## 4. Backlog MCP の接続方式を決める

MCP サーバーには主に以下の接続方式がある。

| 方式 | 使う場面 |
|---|---|
| `stdio` | ローカルで MCP サーバーを起動する場合 |
| `http` | リモートの MCP サーバーへ接続する場合 |

Backlog MCP は、社内で使う MCP サーバーの配布形態に合わせて選ぶ。

初回セットアップ会では、個人の秘密情報をリポジトリに入れないため、まずは `--scope local` または `--scope user` で登録するのがよい。

| scope | 用途 | 保存先 |
|---|---|---|
| `local` | 今開いているプロジェクトだけで使う | `~/.claude.json` |
| `user` | 自分の全プロジェクトで使う | `~/.claude.json` |
| `project` | チームで共有する | `.mcp.json` |

Backlog API キーを含む設定は、原則として `project` scope にしない。

## 5. Backlog API キーを環境変数に入れる

ターミナルで一時的に設定する場合。

```bash
export BACKLOG_SPACE_URL="https://example.backlog.com"
export BACKLOG_API_KEY="your-backlog-api-key"
```

この方法は、ターミナルを閉じると消える。

セットアップ会では、まずこの一時設定で動作確認するのが安全。

毎回使う場合は、各自の shell 設定に入れる。ただし、API キーをリポジトリ内のファイルに書かない。

## 6. Backlog MCP を Claude Code に追加する

### stdio 型の例

Backlog MCP サーバーをローカルで起動するタイプの場合。

以下は形の例。`<backlog-mcp-package>` は、実際に使う MCP サーバーのパッケージ名に差し替える。

```bash
claude mcp add --transport stdio backlog --scope local -- \
  npx -y <backlog-mcp-package>
```

もし MCP サーバーが引数で URL や API キーを受け取る仕様なら、README に合わせて指定する。

例。

```bash
claude mcp add --transport stdio backlog --scope local -- \
  npx -y <backlog-mcp-package> \
  --space-url "$BACKLOG_SPACE_URL" \
  --api-key "$BACKLOG_API_KEY"
```

環境変数を読むタイプなら、`export` 済みの状態で追加する。

### http 型の例

社内や公式のリモート MCP サーバーに接続するタイプの場合。

```bash
claude mcp add --transport http backlog --scope local https://example.com/mcp
```

Bearer token をヘッダーで渡す仕様なら、以下のような形になる。

```bash
claude mcp add --transport http backlog --scope local https://example.com/mcp \
  --header "Authorization: Bearer $BACKLOG_API_KEY"
```

実際の URL、ヘッダー名、認証方式は、利用する Backlog MCP サーバーの README に合わせる。

## 7. MCP の接続状態を確認する

Claude Code を起動する。

```bash
claude
```

Claude Code 内で以下を実行する。

```text
/mcp
```

`backlog` が表示され、接続済みになっていることを確認する。

OAuth 認証が必要な MCP サーバーの場合は、`/mcp` からブラウザ認証を進める。

コマンドラインから認証できる MCP サーバーなら、以下を使う。

```bash
claude mcp login backlog
```

## 8. Backlog の課題を読めるか確認する

最初は read 系の依頼だけにする。

```text
Backlog MCP を使って、参加しているプロジェクトの一覧を取得して
```

```text
Backlog MCP を使って、プロジェクト ABC の未完了課題を 5 件だけ取得して
```

```text
Backlog MCP を使って、課題 ABC-123 の要約をして
```

ここで、Claude Code が MCP tool の実行確認を出す場合がある。

初回は tool 名を確認し、Backlog の読み取り系であることを確認してから許可する。

## 9. permissions を設定する

Backlog MCP は外部サービスに触るため、read 系から始める。

`~/.claude/settings.json` またはプロジェクトの設定に、考え方として以下を入れる。

実際の MCP tool 名は `/mcp` や実行確認画面で確認し、環境に合わせて調整する。

```json
{
  "permissions": {
    "allow": [
      "mcp__backlog__*get*",
      "mcp__backlog__*list*",
      "mcp__backlog__*search*",
      "mcp__backlog__*read*"
    ],
    "ask": [
      "mcp__backlog__*create*",
      "mcp__backlog__*update*",
      "mcp__backlog__*comment*"
    ],
    "deny": [
      "mcp__backlog__*delete*",
      "mcp__backlog__*remove*"
    ]
  }
}
```

初回セットアップ会では、以下の方針にする。

- 課題の検索、取得、一覧は許可
- コメント追加、課題更新、課題作成は確認
- 削除系は拒否

## 10. 動作確認のゴール

以下ができればセットアップ完了。

- `claude --version` が通る
- `claude doctor` が大きな問題なく通る
- `claude` でログイン済みになる
- `/mcp` に `backlog` が表示される
- Backlog の課題を 1 件読める
- Backlog の write 系操作は勝手に実行されない

## よくある詰まり

### `claude` コマンドが見つからない

ターミナルを開き直す。

それでもだめなら `PATH` を確認する。

```bash
echo "$PATH"
which claude
```

### Backlog MCP が接続できない

確認すること。

- `BACKLOG_SPACE_URL` が正しいか
- `BACKLOG_API_KEY` が正しいか
- API キーに対象プロジェクトを読む権限があるか
- MCP サーバーの README に書かれた環境変数名と一致しているか
- 社内プロキシや VPN が必要ではないか

### `/mcp` に出てこない

Claude Code を再起動する。

それでも出ない場合は、追加コマンドをもう一度確認する。

```bash
claude mcp add --help
```

### 読み取りはできるが更新もできてしまいそう

permissions を先に入れる。

初回は read 系だけ使う。

コメント追加や課題更新は、セットアップ会では実行しない。

## 当日配布用の最短コマンド列

Backlog MCP サーバーが `stdio` 型で、環境変数から認証情報を読む場合の例。

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
claude doctor

export BACKLOG_SPACE_URL="https://example.backlog.com"
export BACKLOG_API_KEY="your-backlog-api-key"

cd /path/to/your/project

claude mcp add --transport stdio backlog --scope local -- \
  npx -y <backlog-mcp-package>

claude
```

Claude Code 内で実行する。

```text
/mcp
Backlog MCP を使って、課題 ABC-123 の内容を要約して
```

## 参考

- Claude Code setup: https://code.claude.com/docs/en/setup
- Claude Code MCP: https://code.claude.com/docs/en/mcp
- Claude Code settings: https://code.claude.com/docs/en/settings
- Claude Code permissions: https://code.claude.com/docs/en/permissions
