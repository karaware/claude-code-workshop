# AI を飼い慣らせ

## サブタイトル

Claude Code を制御して共有環境の事故を防ぐ

## 背景

部署内で Claude Code の勉強会を継続して実施してきた。

過去のテーマ:

1. Claude Code で手順書自動作成
2. Claude Code で構成図自動作成、自作ノウハウ蓄積ツール紹介

毎回アンケートを取った結果、以下の不安が多かった。

- Claude Code によって環境を破壊するのが怖い
- 機密情報の取り扱いミスでインシデントになるのが怖い
- 自走させたいが、任せきるのは怖い

ここでいう「環境破壊」は、ローカル PC の破壊というより、AWS アカウント、Google Drive、GitHub、Slack、Jira、社内 API など、他者と共有しているリモート環境の破壊を主に想定する。

## 勉強会の目的

Claude Code を禁止するのではなく、制御して安全に使う。

最終的には、permissions や hooks でやってほしくないことを制御したうえで、auto mode で安全に自走させることを目指す。

## 一番伝えたいメッセージ

Claude Code を安全に使うとは、Claude を信じることではない。

Claude が触れる権限・道具・環境を先に設計して、その範囲内で自走させること。

## 制御レイヤーの全体像

| レイヤー | 役割 | 勉強会での扱い |
|---|---|---|
| CLAUDE.md | Claude の行動方針を誘導する。しつけ。 | 補助 |
| permissions | 何を許可・確認・拒否するかを決める。 | 主役 |
| permission mode | どの程度ユーザー確認を求めるかを決める。 | 主役 |
| hooks | 実行前チェック、監査ログ、社内ルール実装。 | 主役 |
| MCP 制御 | Google Drive / GitHub / Slack など外部サービス操作の入口を制御する。 | 主役 |
| sandbox | ローカル Bash 実行のファイル・ネットワーク到達範囲を制限する。 | 補足 |
| managed settings | 組織・事業部側で強制する設定。 | 概要のみ |

## 比喩

| 仕組み | 比喩 |
|---|---|
| CLAUDE.md | しつけ |
| permissions | リード |
| hooks | 門番 |
| MCP 制御 | 外に出るドアの鍵 |
| sandbox | 室内の柵 |
| permission mode | 放し飼いレベル |
| managed settings | 会社・事業部のルール |

## 事故の種類

### ローカル PC 破壊

例:

- `rm -rf`
- 不要なファイル削除
- ローカル設定ファイルの破壊
- 想定外のネットワークアクセス

主な対策:

- permissions
- hooks
- sandbox

### 共有リモート環境破壊

例:

- `aws s3 rm`
- `aws iam delete-role`
- `terraform apply`
- `terraform destroy`
- `kubectl delete`
- `gh repo delete`
- Google Drive の共有ファイル削除
- Slack への誤投稿
- MCP 経由の外部サービス操作

主な対策:

- permissions
- hooks
- MCP 制御
- 外部サービス側の権限設計
- permission mode

## permissions

Claude Code に「何をしてよいか・ダメか」を決める中心。

基本方針:

- 調査系、read 系は許可する
- 更新、削除、適用、投稿、共有変更は拒否または確認にする
- 本番環境への write 操作は原則拒否する

### 設定イメージ

```json
{
  "permissions": {
    "allow": [
      "Bash(aws sts get-caller-identity)",
      "Bash(aws s3 ls*)",
      "Bash(aws cloudformation describe-*)",
      "Bash(terraform plan*)",
      "Bash(kubectl get *)",
      "Bash(kubectl describe *)"
    ],
    "deny": [
      "Bash(aws * delete*)",
      "Bash(aws * remove*)",
      "Bash(aws * put*)",
      "Bash(aws * update*)",
      "Bash(aws * create*)",
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)",
      "Bash(kubectl delete *)",
      "Bash(kubectl apply *)",
      "Bash(kubectl patch *)",
      "Bash(kubectl scale *)",
      "Bash(gh repo delete*)",
      "Bash(rm -rf *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.aws/**)",
      "Read(~/.ssh/**)"
    ]
  }
}
```

## permission mode

permission mode は、ツール実行時にどの程度ユーザー確認を求めるかを決める。

今回の勉強会ではかなり重要。

目指したい姿:

1. 最初は default mode で慎重に使う
2. permissions / hooks / MCP 制御で危険操作を潰す
3. read-only 作業や限定された作業から auto mode を使う
4. 安全な範囲内で Claude Code を自走させる

重要な考え方:

> auto mode は「なんでもやらせるモード」ではない。
>
> 先に permissions / hooks / MCP 制御でやってはいけないことを潰したうえで、残った安全な作業を自走させるためのモード。

悪い設計:

```text
auto mode にすれば Claude が危険操作を判断してくれる
```

良い設計:

```text
危険操作は permissions / hooks / MCP allowlist で先に潰す
そのうえで auto mode に任せる
```

## hooks

hooks は、部署ルールをコード化する場所。

特に重要なのは `PreToolUse` と `PostToolUse`。

### PreToolUse

ツール実行前にチェックし、危険な操作を止める。

止めたい例:

- `aws` で `delete` / `put` / `update` / `create` を含むコマンド
- `terraform apply`
- `terraform destroy`
- `kubectl delete`
- `kubectl apply`
- `kubectl patch`
- `kubectl scale`
- `gh repo delete`
- Google Drive の delete / update 系 MCP
- Slack への post 系 MCP
- 本番っぽい名前を含む操作
  - `prod`
  - `production`
  - `prd`
  - `本番`

### PostToolUse

実行後に監査ログを残す。

記録したい例:

- 実行日時
- 実行コマンド
- 対象リポジトリ
- 作業ディレクトリ
- 使用した MCP tool
- 成功 / 失敗

## MCP 制御

MCP は、Claude Code から外部ツール・外部サービスを使うための入口。

Google Drive、GitHub、Slack、Jira、AWS などに接続する場合、Claude Code にその人の手足を増やすのと同じ。

基本方針:

- MCP は便利な拡張ではなく、外部サービスへの操作権限として扱う
- deny by default を基本にする
- 必要な read 系 tool だけを許可する
- update / delete / post / share / merge などは慎重に扱う

### 設定イメージ

実際の MCP tool 名は MCP サーバーによって変わるため、以下は考え方の例。

```json
{
  "permissions": {
    "allow": [
      "mcp__github__get_issue",
      "mcp__github__list_pull_requests",
      "mcp__google_drive__search",
      "mcp__google_drive__read_file"
    ],
    "deny": [
      "mcp__github__merge_pull_request",
      "mcp__github__delete_repository",
      "mcp__google_drive__delete_file",
      "mcp__google_drive__update_file",
      "mcp__slack__post_message"
    ]
  }
}
```

## 外部サービス側の権限設計

Claude Code の中で制御する以前に、外部サービス側の権限を絞ることが重要。

### AWS

NG:

- 普段使いの強い IAM 権限で Claude Code を使う

OK:

- 読み取り専用 role
- 検証環境専用 role
- 短時間の一時クレデンシャル
- 本番 write 権限なし

### Google Drive

NG:

- マイドライブ全体に強い権限を持つ MCP を接続する

OK:

- 特定フォルダだけ
- 読み取り中心
- 削除・共有設定変更は不可

### GitHub

NG:

- repo 全体 write / admin 権限の token
- main 直接 push 可能な状態で自由に動かす

OK:

- 対象 repo 限定
- read 中心
- PR 作成まで
- main 直接 push 禁止
- branch protection を使う

## sandbox

sandbox は、Claude Code がローカルで実行する Bash コマンドを隔離された環境で動かすための仕組み。

ローカルファイルの破壊や、想定外のネットワークアクセスを抑えるには有効。

ただし、AWS / Google Drive / GitHub / Slack などの共有リモート環境を守る本命ではない。

なぜなら、それらの事故はローカルファイル破壊ではなく、認証済み API を通じたリモート操作として起きるから。

そのため、共有リモート環境の事故防止では、permissions / hooks / MCP 制御 / 外部サービス側の権限設計が重要。

## managed settings

managed settings は、組織や事業部側で強制する設定。

個人の `~/.claude/settings.json` より上位の制御として扱われる。

今回の部署勉強会では概要だけ触れる。

伝えること:

- 個人設定だけでは、各自が勝手に緩い設定にできる
- 組織・事業部として守らせたいルールは managed settings 側で制御する
- 部署内では、その上にプロジェクト設定や hooks を重ねる

## CLAUDE.md

CLAUDE.md は Claude の行動方針を誘導するためのもの。

例:

```md
- 本番環境に対する変更は禁止
- terraform apply は実行せず、plan までにする
- .env や秘密鍵を読まない
- 破壊的操作の前に必ず確認する
```

ただし、これは強制力のある制御ではない。

CLAUDE.md はしつけであり、permissions / hooks / sandbox / MCP 制御の代わりにはならない。

## 勉強会構成案

1. 前回までの振り返り
   - 手順書自動作成
   - 構成図自動作成
   - ノウハウ蓄積ツール

2. アンケートで見えた不安
   - 環境破壊が怖い
   - 機密情報の扱いが怖い
   - 自走させたいが、任せきるのは怖い

3. 今回のゴール
   - Claude Code を禁止するのではなく、制御して使う
   - 最終的には auto mode で安全に自走させる

4. 事故の種類を分解する
   - ローカル PC 破壊
   - AWS / Google Drive / GitHub / Slack など共有リモート環境破壊
   - 機密情報読み取り
   - 外部投稿・外部共有

5. Claude Code の制御レイヤー
   - CLAUDE.md
   - permissions
   - permission mode
   - hooks
   - MCP
   - sandbox
   - managed settings

6. permissions
   - allow / ask / deny
   - read は許可、write/delete/apply は止める
   - AWS / kubectl / terraform の例

7. permission mode
   - default
   - acceptEdits
   - auto
   - dontAsk
   - bypassPermissions
   - auto mode は制御後に使う

8. hooks
   - PreToolUse で危険操作を止める
   - PostToolUse でログを残す
   - UserPromptSubmit で機密情報っぽい入力を検知する

9. MCP
   - MCP は外部サービス操作権限
   - Google Drive / GitHub / Slack 連携の注意点
   - deny by default で必要な tool だけ許可

10. sandbox
    - ローカル Bash の柵
    - 今回の本命ではないが、知っておく価値はある

11. managed settings
    - 事業部側で強制する設定
    - 個人では上書きできない領域

12. 推奨運用
    - 最初は default mode
    - read-only 作業から始める
    - dangerous command は deny
    - hooks で部署ルールを足す
    - 慣れてきたら auto mode

## デモ案

### デモ 1: 読み取りは許可、変更は拒否

- `aws sts get-caller-identity` は通す
- `aws s3 ls` は通す
- `aws s3 rm` は止める

### デモ 2: Terraform

- `terraform plan` は通す
- `terraform apply` は止める
- `terraform destroy` は止める

### デモ 3: Kubernetes

- `kubectl get pods` は通す
- `kubectl describe pod` は通す
- `kubectl delete pod` は止める
- `kubectl apply` は止める

### デモ 4: MCP

- GitHub issue の読み取りは通す
- PR merge は止める
- Google Drive の検索・読み取りは通す
- Google Drive の削除・更新は止める
- Slack への投稿は止める

### デモ 5: hooks による監査ログ

- Claude Code が実行したコマンドをログに残す
- どの MCP tool を使ったかをログに残す

## 参考リンク

- Claude Code settings: https://code.claude.com/docs/en/settings
- Claude Code permissions: https://code.claude.com/docs/en/permissions
- Claude Code permission modes: https://code.claude.com/docs/en/permission-modes
- Claude Code hooks: https://code.claude.com/docs/en/hooks
- Claude Code sandboxing: https://code.claude.com/docs/en/sandboxing
