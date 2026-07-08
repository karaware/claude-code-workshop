# Claude Code auto mode classifier 説明メモ

## このファイルの目的

Claude Code の `auto mode` に出てくる `classifier` について、勉強会で説明できるように整理する。

20分勉強会では、classifier を深掘りしすぎず、以下の理解を持ち帰ってもらう。

```text
classifier は、auto mode 中に Claude Code のツール実行を判定する追加の安全ゲート。
ただし万能ではないので、permissions の deny / ask と組み合わせる。
```

## 参照した公式ドキュメント

- Configure auto mode: https://code.claude.com/docs/en/auto-mode-config
- Permission modes: https://code.claude.com/docs/en/permission-modes
- Permissions: https://code.claude.com/docs/en/permissions

## ざっくり結論

classifier は、`auto mode` 中に Claude Code がツールを実行しようとしたとき、その操作が安全そうか危険そうかを判定する仕組み。

通常の manual / default mode では、Claude Code が危険そうな操作をしようとするとユーザーに確認プロンプトが出る。

一方、auto mode では routine な確認プロンプトを減らすため、代わりに classifier がツール実行を見て、以下のような操作を止める。

- irreversible な操作
- destructive な操作
- 自分たちの環境外に向けた操作
- 情報流出につながる操作
- 本番環境や共有インフラへの危険な操作

ただし、classifier は安全を保証する魔法ではない。

公式ドキュメントでも、`permissions.deny` や明示的な `ask` ルールは classifier より前に評価されると説明されている。

そのため、絶対に止めたいものは classifier 任せにせず、permissions 側で止める。

## 一言でいうと

```text
classifier は、auto mode の自走を支える安全判定役。
ただし、最終防衛ラインではなく、permissions の後ろにいる第二のゲート。
```

別の言い方をすると、

```text
permissions = 明示的なルール
classifier = 文脈を見て危険そうな操作を判定する門番
```

## auto mode における処理の順番

勉強会では、以下の順番で説明するとわかりやすい。

```text
Claude Code がツールを実行しようとする
↓
permissions.deny に当たるか？
  → 当たるなら拒否
↓
permissions.ask に当たるか？
  → 当たるなら人間に確認
↓
auto mode の classifier が判定する
  → 危険そうならブロックまたは確認に戻す
  → 問題なさそうなら実行
```

重要なのは、classifier よりも `deny` / 明示的な `ask` の方が前にあること。

つまり、

```text
絶対にダメな操作は classifier に判断させない。
先に permissions.deny で止める。
```

という設計が安全。

## classifier が見ているもの

公式ドキュメント上、classifier は以下の情報を使う。

### 1. ツール実行の内容

例:

- どの tool を使うか
- Bash ならどんなコマンドか
- MCP ならどの MCP tool か
- どのファイルやリモート環境に触ろうとしているか

### 2. 会話の文脈

例:

- ユーザーが何を依頼したか
- ユーザーが明示的に許可した操作か
- ユーザーが「まだ push しないで」「deploy しないで」と言っていないか

ただし、会話の文脈は永続的な設定ではない。

context compaction などで文脈から落ちる可能性があるため、強いルールは permissions に書く。

### 3. CLAUDE.md

classifier は、Claude 本体が読む `CLAUDE.md` の内容も読む。

たとえば、以下のようなルールは Claude と classifier の両方に意図を伝える。

```md
- production 環境への deploy は実行しない
- terraform apply / destroy は必ずユーザー確認を挟む
- main / master への直接 push はしない
- Slack 投稿はユーザーに確認してから行う
```

ただし、`CLAUDE.md` は強制力のある制御ではない。

止めたいものは `permissions.deny` や `permissions.ask` に書く。

### 4. autoMode 設定

classifier は `autoMode` 設定を読む。

特に重要なのは `autoMode.environment`。

これは classifier に対して、どの repo、bucket、domain、service が自分たちの信頼境界の内側なのかを教える設定。

例:

```json
{
  "autoMode": {
    "environment": [
      "$defaults",
      "Organization: Example Corp. Primary use: infrastructure automation",
      "Cloud provider(s): AWS",
      "Source control: GitHub org github.example.com/example-corp",
      "Trusted internal domains: *.internal.example.com",
      "Trusted cloud buckets: s3://example-build-artifacts",
      "Protected IaC scopes: production AWS accounts and terraform/prod directories"
    ]
  }
}
```

公式ドキュメントでは、`autoMode.environment` の entries は regex や tool pattern ではなく、自然言語で書くとされている。

## autoMode.environment が必要な理由

classifier は、何が「社内の通常操作」で、何が「外部への危険な操作」なのかを完全には知らない。

たとえば、以下の違いは会社ごとに異なる。

```text
- 社内 GitHub org
- 信頼してよい S3 bucket
- 社内 API domain
- CI/CD server
- artifact registry
- production namespace
```

そのため、`autoMode.environment` で信頼境界を教える。

何も設定しない場合、classifier は conservative に動く。

公式ドキュメントでは、既定では working directory と current repo の configured remotes だけを信頼する、と説明されている。

つまり、社内 GitHub org や team cloud bucket への操作でも、environment に書いていないとブロックされる可能性がある。

## classifier がブロックしようとするもの

公式ドキュメントでは、classifier は以下のような操作をブロックする方向で動く。

勉強会では、全部を説明せず、インフラ部門に刺さるものだけ出すとよい。

### 代表例

- `curl | bash` のような、ダウンロードしたコードの実行
- sensitive data の外部送信
- production deploy
- migration
- cloud storage の大量削除
- IAM や repository 権限の変更
- 共有インフラの変更
- force push
- `terraform destroy`
- destroy を含む apply
- DNS / TLS certificate の変更
- human approval のない PR merge
- CI check の無効化
- deploy bot に対する `/deploy` や `/merge` のようなコメント投稿
- production feature flag の変更
- Kubernetes の危険な変更
- sensitive remote target への interactive shell や port-forward
- live credential / token の transcript や file への出力

## classifier が万能ではない理由

### 1. 会社固有の事情を知らない

classifier は、何が社内で安全な操作かを最初から完全には知らない。

たとえば、以下は組織ごとに違う。

- どの AWS account が dev か prod か
- どの S3 bucket が一時領域か
- どの GitHub org が社内か
- どの Slack channel に投稿してよいか
- どの Jira project を更新してよいか

そのため、`autoMode.environment` で context を与える。

### 2. 「許可してよい操作」と「絶対ダメな操作」は人間が決める必要がある

classifier は安全そうか危険そうかを判定するが、最終的な責任境界は人間側で設計する。

例:

```text
.env は絶対に読ませたくない
terraform apply は人間承認を挟みたい
Slack 投稿は確認したい
PR merge は禁止したい
```

このような組織・部署のルールは、permissions に書く。

### 3. allow rule が先に通る場合がある

公式ドキュメントでは、狭い Bash / PowerShell allow rule は auto mode 中も classifier より先に解決されることがあると説明されている。

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

このような allow は便利だが、想定外の引数や script 経由で危険な動作につながる可能性がある。

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

## permissions と classifier の使い分け

### permissions.deny

絶対に実行させたくないもの。

例:

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.ssh/**)",
      "Bash(gh repo delete*)",
      "mcp__github__delete_repository",
      "mcp__google_drive__delete_file"
    ]
  }
}
```

### permissions.ask

実行する可能性はあるが、人間の承認を挟みたいもの。

例:

```json
{
  "permissions": {
    "ask": [
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)",
      "Bash(kubectl apply *)",
      "Bash(kubectl delete *)",
      "mcp__slack__post_message",
      "mcp__github__create_pull_request"
    ]
  }
}
```

### classifier

事前に全部をルール化しきれない操作について、文脈を見て危険そうかを判定する。

例:

- そのコマンドはユーザーの依頼範囲内か
- 外部サービスに送ろうとしているデータは信頼境界の外か
- 本番っぽい対象を変更しようとしていないか
- hostile content に誘導されていないか
- force push や大量削除のような危険操作ではないか

## 勉強会での説明例

### 短い説明

```text
classifier は、auto mode 中に Claude Code の操作を見て、危険そうなら止める安全判定役です。
ただし、絶対に止めたいものは classifier 任せにせず、permissions.deny に書きます。
```

### 少し詳しい説明

```text
auto mode では、通常の確認プロンプトを減らす代わりに、classifier がツール実行前に安全性を判定します。
classifier は、操作内容、会話の文脈、CLAUDE.md、autoMode.environment などを見て、
それがユーザーの依頼範囲内か、外部への情報流出にならないか、本番や共有環境を壊さないかを判断します。

ただし、classifier は会社固有の事情を完全には知りません。
なので、.env は deny、terraform apply は ask、MCP の delete / merge は deny のように、
人間側で境界を設計する必要があります。
```

## 比喩で説明するなら

```text
permissions.deny = 絶対に開かない鍵付きドア
permissions.ask = 開ける前に人間へ確認するドア
classifier = 通ろうとしている人や荷物を見て止める警備員
autoMode.environment = 警備員に渡す社内地図
CLAUDE.md = 警備員と Claude への申し送り
```

または、今回の勉強会タイトルに合わせるなら、

```text
permissions = リード
classifier = 散歩中に危ない方向へ行かないか見る見張り
autoMode.environment = 散歩してよい範囲の地図
```

## 20分勉強会で使うなら

classifier の説明は長くしすぎない。

おすすめは 1 スライドだけ。

### スライド案

```text
classifier とは？

auto mode 中の安全判定役

見るもの:
- 実行しようとしているツール
- 会話の文脈
- CLAUDE.md
- autoMode.environment

ただし:
- 絶対にダメなものは permissions.deny
- 人間承認したいものは permissions.ask
```

### 話すフレーズ

```text
auto mode は、確認を全部飛ばすモードではありません。
通常の確認プロンプトを減らす代わりに、classifier が裏側で安全性を判定します。
ただし classifier は万能ではないので、.env のように絶対読ませたくないものは deny、terraform apply のように承認したいものは ask にします。
```

## 既存資料への差し込み位置

`20min-workshop-outline.md` では、以下の位置に入れるとよい。

```text
3:00 - 6:00 permissions の基本
```

この中で、`allow / ask / deny` を説明した後に、30秒だけ classifier に触れる。

```text
permissions が明示的なルール。
auto mode では、その後ろで classifier が安全性を判定する。
```

深掘りは `auto-mode-summary.md` とこのファイルに逃がす。
