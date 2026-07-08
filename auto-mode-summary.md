# Claude Code auto mode 調査メモ

## このファイルの目的

Claude Code の `auto mode` について、勉強会で使えるように要点を 1 ファイルにまとめる。

20分勉強会では、以下の流れで話す想定。

```text
auto mode は作業を加速する本丸
↓
ただし安全を保証するものではない
↓
permissions / autoMode 設定 / MCP 制御で範囲を設計する
↓
そのうえで安全に作業を加速する
```

## 参照した公式ドキュメント

- Permission modes: https://code.claude.com/docs/en/permission-modes
- Configure auto mode: https://code.claude.com/docs/en/auto-mode-config
- Permissions: https://code.claude.com/docs/en/permissions
- Settings: https://code.claude.com/docs/en/settings

## ざっくり結論

`auto mode` は、Claude Code による作業を大きく加速するための permission mode。

通常は、Claude Code がファイル編集・Bash 実行・ネットワークアクセスなどを行うたびにユーザー確認が入る。

`auto mode` では、通常の確認プロンプトを減らし、別の classifier がツール実行前に安全性を判定する。

ただし、`auto mode` は安全を保証するものではない。

公式ドキュメントでも research preview とされており、機密情報・本番環境・共有リモート環境に関わる作業では、permissions や autoMode 設定で事前に範囲を絞ることが重要。

## auto mode とは

Claude Code の permission mode の 1 つ。

公式ドキュメント上の位置づけは以下。

| mode | 概要 | 向いている用途 |
|---|---|---|
| `default` | 読み取り中心。必要に応じて確認する。CLI では Manual と表示される | 初心者、慎重な作業 |
| `acceptEdits` | ファイル編集や一部のファイル操作を自動許可する | コード修正をレビューしながら進める |
| `plan` | 読み取り・調査・計画まで。編集はしない | 変更前の調査 |
| `auto` | 背景の安全チェック付きで多くの操作を自動実行する | 長い作業、確認疲れの軽減 |
| `dontAsk` | 事前許可されたツール以外は自動拒否する | CI、ロックダウン環境 |
| `bypassPermissions` | ほぼすべての確認をスキップする | 隔離されたコンテナ・VM のみ |

20分勉強会では、細かい mode 比較よりも以下を強調する。

```text
auto mode は「なんでも許可するモード」ではない。
通常の確認プロンプトを減らしつつ、背景の classifier が危険そうな操作を判定するモード。
```

## 何がうれしいか

### 1. 確認プロンプトが減る

Claude Code にまとまった作業を依頼したとき、途中で毎回確認が入ると手が止まる。

`auto mode` は、その確認を減らし、Claude Code に長めの作業を進めさせやすくする。

### 2. 自走させやすい

勉強会の文脈では、ここが一番重要。

Claude Code による作業加速の本丸は、単発の質問回答ではなく、ある程度まとまった作業を自走して進めてもらうこと。

`auto mode` はそのための中心機能として紹介できる。

### 3. bypassPermissions より現実的

`bypassPermissions` は強力だが危険。

`auto mode` は、何でもスキップするのではなく、背景の安全判定を挟むため、`bypassPermissions` より現実的な選択肢として説明できる。

## 何が怖いか

`auto mode` は便利だが、怖さもある。

特にインフラ・社内利用では以下が問題になる。

### 1. 機密情報を読んでしまう

例:

- `.env`
- `.env.*`
- `secrets/**`
- `~/.aws/**`
- `~/.ssh/**`

公式ドキュメントでは、auto mode の既定動作として `.env` の読み取りや、対応する API への credential 送信が許可される旨が説明されている。

そのため、`.env` を読ませたくない場合は、auto mode ではなく `permissions.deny` で明示的に止めるのが重要。

### 2. 共有リモート環境を変更してしまう

例:

- `terraform apply`
- `terraform destroy`
- `aws s3 rm`
- `kubectl delete`
- GitHub の main への push
- Google Drive の共有ファイル更新・削除

公式ドキュメントでは、auto mode は本番 deploy、migration、大量削除、IAM や repo 権限変更、共有インフラ変更などを既定でブロックする方向だが、これは安全保証ではない。

### 3. MCP 経由で外部サービスを操作してしまう

MCP は Claude Code に外部サービスを操作する手足を生やすもの。

例:

- Google Drive を読む・更新する・削除する
- GitHub の Issue / PR を操作する
- Slack に投稿する
- Jira のチケットを更新する

MCP は便利だが、外部サービス操作権限そのものとして扱う必要がある。

## auto mode の安全性の考え方

公式ドキュメント上、auto mode は以下のような仕組みで安全性を高めようとしている。

- 背景の classifier が実行前にアクションを確認する
- ユーザーの依頼範囲を超える操作をブロックする
- 未認識のインフラを対象にする操作をブロックする
- hostile content によって誘導されたように見える操作をブロックする
- 明示的な `ask` ルールは auto mode 中でも確認を強制する
- `deny` ルールは auto mode 中でも先に効く

重要なのは、auto mode の classifier は「第二のゲート」であり、`permissions.deny` や明示的な `ask` ルールの代わりではないこと。

## permission rules との関係

Claude Code の permissions では、以下の 3 種類を設定できる。

| 種類 | 意味 |
|---|---|
| `allow` | 指定したツール実行を自動許可する |
| `ask` | 指定したツール実行の前に確認する |
| `deny` | 指定したツール実行を拒否する |

公式ドキュメントでは、ルールの評価順は以下。

```text
deny → ask → allow
```

つまり、広い `deny` にマッチした操作は、より狭い `allow` があっても止まる。

勉強会ではこう説明するとよい。

```text
auto mode を安全に使うには、
先に permissions で「絶対ダメ」「人間確認が必要」を決めておく。
```

## auto mode と permissions の使い分け

### deny にすべきもの

「どんな状況でも実行させたくないもの」。

例:

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.ssh/**)",
      "Bash(gh repo delete*)"
    ]
  }
}
```

### ask にすべきもの

「実行する可能性はあるが、人間の承認を挟みたいもの」。

例:

```json
{
  "permissions": {
    "ask": [
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)",
      "Bash(kubectl delete *)",
      "Bash(kubectl apply *)"
    ]
  }
}
```

### allow にしてよいもの

「調査や確認として安全に実行させたいもの」。

例:

```json
{
  "permissions": {
    "allow": [
      "Bash(terraform plan*)",
      "Bash(aws sts get-caller-identity)",
      "Bash(kubectl get *)",
      "Bash(kubectl describe *)"
    ]
  }
}
```

## autoMode 設定

`auto mode` には、通常の `permissions` とは別に `autoMode` 設定がある。

これは、auto mode の classifier に対して、組織内で信頼するリポジトリ・バケット・ドメイン・サービスなどを伝えるための設定。

公式ドキュメントでは、主に以下が紹介されている。

| 設定 | 役割 |
|---|---|
| `autoMode.environment` | 信頼するリポジトリ、バケット、ドメイン、社内サービスなどを自然言語で記述する |
| `autoMode.allow` | soft block に対する例外を追加する |
| `autoMode.soft_deny` | ユーザーの明示的意図で解除可能なブロックを追加する |
| `autoMode.hard_deny` | ユーザー意図や allow で解除できない境界を追加する |
| `autoMode.classifyAllShell` | auto mode 中、すべての shell command を classifier に通す |

## autoMode.environment

`autoMode.environment` は、auto mode に「自社にとって何が内側で、何が外側か」を教える設定。

公式ドキュメントでは、entries は regex や tool pattern ではなく、自然言語で書くとされている。

例:

```json
{
  "autoMode": {
    "environment": [
      "$defaults",
      "Organization: Example Corp. Primary use: infrastructure automation",
      "Source control: GitHub org github.example.com/example-corp",
      "Cloud provider(s): AWS",
      "Trusted cloud buckets: s3://example-build-artifacts, s3://example-dev-logs",
      "Trusted internal domains: *.internal.example.com, api.internal.example.com",
      "Key internal services: Jenkins at ci.internal.example.com, Artifactory at artifacts.internal.example.com",
      "Protected IaC scopes: production AWS accounts and terraform/prod directories"
    ]
  }
}
```

ポイント:

- `"$defaults"` を入れると既定ルールを残したまま追加できる
- `"$defaults"` を入れずに `environment` などを設定すると、その section の既定値を置き換える
- 特に `allow` / `soft_deny` / `hard_deny` では `"$defaults"` を消すと危険な既定ルールを失う可能性がある

## autoMode.classifyAllShell

既定では、狭い Bash allow rule は auto mode 中も classifier の前に解決されることがある。

例:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test*)"
    ]
  }
}
```

このような allow があると、auto mode 中に classifier を通さず許可される範囲が残る可能性がある。

より安全側に倒すなら、以下を設定する。

```json
{
  "autoMode": {
    "classifyAllShell": true
  }
}
```

これにより、auto mode 中はすべての Bash / PowerShell command を classifier に通す。

ただし、classifier call が増えるため、レイテンシやコストとのトレードオフがある。

## 使い始め方

### CLI で切り替える

セッション中に `Shift+Tab` で permission mode を切り替えられる。

`auto mode` は、アカウント・プラン・モデルなどの条件を満たしている場合に選択肢として現れる。

### 起動時に指定する

```bash
claude --permission-mode auto
```

### デフォルトにする

ユーザー設定で指定する。

```json
{
  "permissions": {
    "defaultMode": "auto"
  }
}
```

注意点:

- 公式ドキュメントでは、`defaultMode: "auto"` は user settings または managed settings で使う想定
- 共有 project settings から repo 自身が auto mode を有効化することはできない
- Bedrock / Google Cloud Agent Platform / Microsoft Foundry / Claude apps gateway では `CLAUDE_CODE_ENABLE_AUTO_MODE` の設定が必要な場合がある

## auto mode が既定でブロックしようとする代表例

公式ドキュメントでは、auto mode の classifier が既定でブロックする対象として、以下のような例が挙げられている。

- `curl | bash` のような、ダウンロードしたコードの実行
- sensitive data の外部送信
- production deploy / migration
- cloud storage の大量削除
- IAM や repo 権限の付与・変更
- 共有インフラの変更
- force push
- `terraform destroy` や destroy を含む apply
- DNS / TLS certificate の変更
- human approval のない PR merge
- CI check の無効化
- deploy bot に対する `/deploy` や `/merge` のようなコメント投稿
- production feature flag の変更
- protected IaC scope への apply
- Kubernetes の DaemonSet や admission webhook 作成
- sensitive remote target への interactive shell や port-forward
- live credential / token の transcript や file への出力
- public registry への迂回 install
- `--insecure` のような safety guard を外す flag

勉強会では、すべてを説明する必要はない。

インフラ部署向けには、以下だけで十分。

```text
- terraform destroy
- production deploy
- IAM / repo 権限変更
- cloud storage 大量削除
- force push
- MCP 経由の外部サービス操作
```

## auto mode で許可されやすい代表例

公式ドキュメントでは、既定で許可されるものとして以下のような例がある。

- working directory 内の local file operation
- lock file や manifest に基づく dependency install
- `.env` の読み取りと、対応する API への credential 送信
- read-only HTTP request
- 開始時の branch または Claude が作成した branch への push
- trusted domain / bucket / service への data 送信

ここで重要なのは、`.env` が既定では許可される側に含まれていること。

そのため、勉強会では `.env deny` のネタとつなげる。

```text
auto mode 側に任せるのではなく、
読ませたくないものは permissions.deny で明示的に止める。
```

## auto mode と MCP

auto mode では、MCP tool も classifier の対象になる。

ただし、MCP は外部サービスを操作する手足なので、auto mode に任せきるのではなく、permissions 側でも制御する方がよい。

例:

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

実際の MCP tool 名は MCP server によって変わる。

勉強会では、設定例そのものより、以下の考え方を伝える。

```text
MCP は便利機能ではなく、外部サービス操作権限。
Claude Code に手足を生やすものとして扱う。
```

## auto mode と CLAUDE.md

auto mode の classifier は、Claude 自身が読む `CLAUDE.md` も参照する。

たとえば、`CLAUDE.md` に以下のような方針を書いておくと、Claude と classifier の両方に意図を伝えられる。

```md
- production 環境への deploy は実行しない
- terraform apply / destroy は必ずユーザー確認を挟む
- .env や secrets 配下のファイルを読まない
- main / master への直接 push はしない
- Slack 投稿はユーザーに確認してから行う
```

ただし、`CLAUDE.md` は強制力のある制御ではない。

ハードに止めたいものは `permissions.deny` または managed settings を使う。

## auto mode と conversation boundary

公式ドキュメントでは、会話内でユーザーが示した境界も classifier が block signal として扱うと説明されている。

例:

```text
- まだ push しないで
- deploy は私が確認するまで待って
- terraform apply はしないで
```

ただし、この境界は永続的な設定ではない。

context compaction によってその発言が文脈から落ちる可能性があるため、強い保証が必要なものは permissions で設定する。

## auto mode が詰まったとき

auto mode で classifier が操作を拒否すると、Claude は別の方法を試す。

繰り返し拒否された場合、auto mode は一時停止し、通常の permission prompt に戻る。

公式ドキュメントでは、拒否された操作は `/permissions` の Recently denied に表示されると説明されている。

また、以下の CLI で設定確認ができる。

```bash
claude auto-mode defaults
claude auto-mode config
claude auto-mode critique
```

| コマンド | 用途 |
|---|---|
| `claude auto-mode defaults` | built-in の environment / allow / soft_deny / hard_deny を確認する |
| `claude auto-mode config` | 現在有効な auto mode 設定を確認する |
| `claude auto-mode critique` | custom rule の曖昧さや重複をレビューしてもらう |

## 勉強会での説明用まとめ

### 一言でいうと

```text
auto mode は、Claude Code に作業を自走させるための便利なモード。
ただし、安全を保証するものではない。
だから、permissions と autoMode 設定で走れる範囲を先に設計する。
```

### 参加者に持ち帰ってほしいこと

```text
1. auto mode は作業加速の本丸
2. でも放し飼いではない
3. .env のような機密情報は deny する
4. terraform apply / destroy のような変更操作は ask にする
5. MCP は外部サービス操作権限として制御する
6. autoMode.environment で自社の trusted boundary を教えられる
```

### 決め台詞

```text
auto mode は放し飼いではない。
柵を作ったうえで、その中を走らせるモード。
```

```text
Claude Code を安全に使うとは、Claude を信じることではない。
Claude が触れる権限・道具・環境を先に設計すること。
```

## 20分勉強会への差し込み方

既存の `20min-workshop-outline.md` には、以下のように差し込む。

### 冒頭

```text
Claude Code の工数削減の本丸は auto mode です。
ただし、auto mode は安全を保証する魔法ではありません。
なので今日は、auto mode を安全に使うための permissions / MCP 制御を扱います。
```

### `.env deny` のところ

```text
公式ドキュメント上、auto mode では .env の読み取りが許可される側に入っています。
だからこそ、読ませたくない場合は permissions.deny で明示的に止める必要があります。
```

### Terraform のところ

```text
auto mode でも terraform destroy などは既定でブロック対象です。
ただし、共有環境に関わる apply / destroy は classifier 任せにせず、ask にして人間承認を挟むのが安全です。
```

### MCP のところ

```text
MCP は Claude Code に外部サービスを操作する手足を生やすものです。
auto mode にすると、その手足も自走対象になります。
だから permissions で read / write / delete / post / merge を分けて考えます。
```

## サンプル設定

20分勉強会で見せるなら、以下くらいに絞る。

```json
{
  "permissions": {
    "defaultMode": "auto",
    "allow": [
      "Bash(terraform plan*)",
      "Bash(aws sts get-caller-identity)",
      "Bash(kubectl get *)",
      "Bash(kubectl describe *)"
    ],
    "ask": [
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)",
      "Bash(kubectl apply *)",
      "Bash(kubectl delete *)",
      "mcp__github__create_pull_request",
      "mcp__slack__post_message"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.ssh/**)",
      "Bash(gh repo delete*)",
      "mcp__github__delete_repository",
      "mcp__github__merge_pull_request",
      "mcp__google_drive__delete_file",
      "mcp__google_drive__update_file"
    ]
  },
  "autoMode": {
    "environment": [
      "$defaults",
      "Organization: Example Corp. Primary use: infrastructure automation",
      "Cloud provider(s): AWS",
      "Source control: GitHub org github.example.com/example-corp",
      "Protected IaC scopes: production AWS accounts and terraform/prod directories"
    ],
    "classifyAllShell": true
  }
}
```

注意:

- 実際の MCP tool 名は環境によって変わる
- `defaultMode: "auto"` を共有 project settings に入れても無視される可能性があるため、user settings または managed settings 側で扱う
- 組織として強制したいルールは managed settings 側に置く
- 20分勉強会では、設定値の丸暗記ではなく、`allow / ask / deny` の考え方を伝える
