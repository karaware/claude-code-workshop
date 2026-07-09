# AI を飼い慣らせ

## サブタイトル

Claude Code を制御して安全に自走させる

## 想定時間

20分

## この資料の位置づけ

詳細メモではなく、20分勉強会で話すための発表用アウトライン。

網羅よりも、以下の流れを伝えることを優先する。

```text
auto mode を安全に使いたい
↓
そのために permissions で触ってよいもの・ダメなもの・確認が必要なものを決める
↓
.env は deny する
↓
terraform apply / destroy は ask にする
↓
MCP は外部サービスに手足を生やすものとして制御する
```

## 今日のゴール

Claude Code には `auto mode` という自走する便利なモードがある。

これは Claude Code による工数削減の本丸。

ただし、何も考えずに auto mode を使うと怖い。

- 機密情報を読んでしまう
- Terraform で共有環境を変更してしまう
- MCP 経由で Google Drive / GitHub / Slack などを操作してしまう

そのため、今回のゴールは以下。

> auto mode を可能な限り安全に使うために、Claude Code の動作を制御する考え方を知る。

## 伝えたいこと

```text
Claude Code を安全に自走させるには、
Claude を信じるのではなく、
Claude が触れる権限を設計する。
```

```text
auto mode は放し飼いではない。
柵を作ったうえで、その中を走らせるモード。
```

## 20分構成

| 時間 | 内容 |
|---|---|
| 0:00 - 3:00 | ゴール説明: auto mode を安全に使う |
| 3:00 - 6:00 | permissions の基本: allow / ask / deny |
| 6:00 - 10:00 | ネタ1: `.env` を deny する |
| 10:00 - 14:00 | ネタ2: `terraform apply` / `destroy` は ask にする |
| 14:00 - 18:00 | ネタ3: MCP は外部サービスに手足を生やすもの |
| 18:00 - 20:00 | まとめ: 柵を作ってから auto mode |

## 0:00 - 3:00 ゴール説明

### 話すこと

Claude Code には auto mode という自走する便利なモードがある。

これは Claude Code による工数削減の本丸。

ただし、何も考えずに自走させると怖い。

特に怖いのはローカル PC だけではなく、他者と共有しているリモート環境。

例:

- AWS アカウント
- Terraform 管理リソース
- Google Drive
- GitHub
- Slack
- Jira
- 社内 API

今回のゴールは、Claude Code を禁止することではない。

制御したうえで、auto mode を可能な限り安全に使うこと。

### スライド見出し案

```text
今日のゴール

auto mode を安全に使う
```

### 話すフレーズ

```text
Claude Code の工数削減の本丸は、自走させることだと思っています。
ただし、何も考えずに auto mode にするのは怖い。
なので今日は、auto mode を安全に使うための制御方法を扱います。
```

## 3:00 - 6:00 permissions の基本

### 話すこと

Claude Code には permissions がある。

permissions では、Claude Code に対して以下を設定できる。

| 種類 | 意味 | 例 |
|---|---|---|
| allow | 自動で許可してよいもの | 調査系コマンド |
| ask | 実行前に人間へ確認させるもの | 変更系コマンド |
| deny | 絶対に実行させないもの | 機密情報の読み取り |

### スライド見出し案

```text
permissions で制御する

allow / ask / deny
```

### 話すフレーズ

```text
Claude Code に「気をつけて」とお願いするのではなく、
何を自動許可し、何を確認し、何を拒否するかを設定します。
```

## 6:00 - 10:00 ネタ1: `.env` を deny する

### 位置づけ

過去に社内ブログで書いたネタを再利用する。

機密情報の取り扱いミスが怖い、というアンケート結果に直接対応する。

### 話すこと

`.env` はアプリケーションの認証情報や API キーが入っていることがある。

Claude Code に「読まないで」と頼むだけでは弱い。

そもそも permissions の deny で読めないようにする。

### 設定例

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  }
}
```

### スライド見出し案

```text
.env は deny

機密情報は「気をつける」ではなく「読ませない」
```

### 話すフレーズ

```text
これは過去の社内ブログでも書いた内容です。
.env のような機密情報は、Claude Code に気をつけてもらうのではなく、
そもそも読めないようにします。
```

## 10:00 - 14:00 ネタ2: `terraform apply` / `destroy` は ask にする

### 位置づけ

共有リモート環境を壊すのが怖い、というアンケート結果に対応する。

### 話すこと

Terraform は Claude Code と相性が良い。

- コードを読める
- 差分を説明できる
- `terraform plan` の結果を要約できる

一方で、`terraform apply` や `terraform destroy` は実際に共有環境を変更する。

そのため、完全禁止の deny ではなく、ask にして人間の確認を挟むのが現実的。

### 設定例

```json
{
  "permissions": {
    "allow": [
      "Bash(terraform plan*)"
    ],
    "ask": [
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)"
    ]
  }
}
```

### 設計思想

```text
plan は Claude Code に任せる。
apply / destroy は人間が判断する。
```

### スライド見出し案

```text
Terraform は plan まで自走

apply / destroy は人間が承認
```

### 話すフレーズ

```text
Terraform は plan までなら Claude Code に任せやすいです。
ただし apply と destroy は共有環境を変える操作なので、
auto mode でも人間の確認を挟む設計にします。
```

## 14:00 - 18:00 ネタ3: MCP は外部サービスに手足を生やすもの

### 位置づけ

Claude Code の便利さと危険さが一気に増えるポイント。

Google Drive / GitHub / Slack など、共有環境への操作と直結する。

### 話すこと

MCP は Claude Code から外部ツール・外部サービスを使うための仕組み。

これは単なる便利機能ではない。

Claude Code に外部サービスを操作する手足を生やすもの。

例:

| MCP 接続先 | できる可能性があること | 怖さ |
|---|---|---|
| Google Drive | ファイル検索、読み取り、更新、削除 | 共有ファイルを壊す |
| GitHub | Issue 読み取り、PR 作成、merge | main に影響する |
| Slack | チャンネル検索、メッセージ投稿 | 誤投稿する |
| Jira | チケット検索、更新 | ステータスを誤変更する |

### 設定イメージ

実際の MCP tool 名は MCP サーバーによって変わるため、考え方の例として扱う。

```json
{
  "permissions": {
    "allow": [
      "mcp__github__get_issue",
      "mcp__github__list_pull_requests",
      "mcp__google_drive__search",
      "mcp__google_drive__read_file"
    ],
    "ask": [
      "mcp__github__create_pull_request",
      "mcp__slack__post_message"
    ],
    "deny": [
      "mcp__github__merge_pull_request",
      "mcp__github__delete_repository",
      "mcp__google_drive__delete_file",
      "mcp__google_drive__update_file"
    ]
  }
}
```

### スライド見出し案

```text
MCP は手足を生やすもの

外部サービス操作権限として扱う
```

### 話すフレーズ

```text
MCP は便利な拡張というより、Claude Code に外部サービスを操作する手足を生やすものです。
だから、何を読ませるか、何を投稿させるか、何を絶対させないかを考える必要があります。
```

## 18:00 - 20:00 まとめ

### 話すこと

今回扱ったこと:

1. auto mode は Claude Code による工数削減の本丸
2. ただし、何も考えずに使うと怖い
3. permissions で allow / ask / deny を設計する
4. `.env` は deny する
5. `terraform apply` / `destroy` は ask にする
6. MCP は外部サービスに手足を生やすものとして制御する

### 最後のメッセージ

```text
auto mode は放し飼いではない。
柵を作ったうえで、その中を走らせるモード。
```

```text
Claude Code を安全に自走させるには、
Claude を信じるのではなく、
Claude が触れる権限を設計する。
```

## スライド構成案

1. AI を飼い慣らせ
   - Claude Code を制御して安全に自走させる

2. 今日のゴール
   - auto mode を安全に使う

3. auto mode は工数削減の本丸
   - でも何も考えずに使うのは怖い

4. 怖さの正体
   - 読まれる
   - 変更される
   - 外部サービスを操作される

5. permissions で制御する
   - allow / ask / deny

6. `.env` は deny
   - 機密情報は読ませない

7. Terraform は plan まで自走
   - apply / destroy は ask

8. MCP は手足を生やすもの
   - 外部サービス操作権限として扱う

9. まとめ
   - 柵を作ってから auto mode

## 補足ネタ候補

時間が余った場合、または次回に回す候補。

### hooks

permissions は静的なルール。

hooks は社内ルールをコードで足せる門番。

例:

- 本番っぽい文字列を含む操作を止める
- 実行コマンドを監査ログに残す
- 特定の MCP tool 実行時に追加チェックする

### sandbox

sandbox はローカル Bash 実行の柵。

ローカルファイル破壊や想定外のネットワークアクセスを抑えるには有効。

ただし、今回の主対象である AWS / Google Drive / GitHub / Slack などの共有リモート環境を守る本命ではない。

### managed settings

事業部や組織側で強制する設定。

個人が勝手に緩い設定へ変えられないようにするための仕組み。

今回は概要だけでよい。
