# classifier / permissions / hooks の役割分担

## このファイルの目的

Claude Code の `auto mode` を安全に使うために、`classifier`、`permissions`、`hooks` の役割分担を整理する。

20分勉強会では、以下を伝えるための補助資料として使う。

```text
classifier は Claude Code 側の内蔵安全判定。
人間が直接しっかり制御する本命は permissions と hooks。
```

## 結論

大筋として、以下の理解でよい。

```text
classifier は Claude Code 側が持つ安全判定。
人間が主に制御する手段は permissions と hooks。
```

ただし、正確には `autoMode` 設定や `CLAUDE.md` によって、classifier に判断材料を与えることはできる。

そのため、整理すると以下。

| 仕組み | 位置づけ | 人間による制御の強さ |
|---|---|---|
| permissions | 明示的な許可・確認・拒否ルール | 強い |
| hooks | 独自ルールをコードで差し込む仕組み | 強い |
| classifier | auto mode 中の Claude Code 側の安全判定 | 直接制御は弱い |
| autoMode 設定 | classifier に環境情報や追加ルールを与える | 中くらい |
| CLAUDE.md | Claude と classifier への申し送り | 弱い |

## それぞれの役割

### classifier

`auto mode` 中に、Claude Code が実行しようとしている操作を見て、安全そうか危険そうかを判定する。

たとえば、以下のような操作を危険と見なす可能性がある。

- irreversible な操作
- destructive な操作
- ユーザーの依頼範囲を超える操作
- 信頼境界の外に情報を送る操作
- production deploy
- `terraform destroy`
- IAM や repository 権限変更
- cloud storage の大量削除
- force push
- human approval のない PR merge

ただし、classifier は Claude Code 側の内蔵判定であり、人間が中身を直接プログラムするものではない。

つまり、以下のような指定は基本的にできない。

```text
この条件なら classifier は必ず許可する
この条件なら classifier は必ず拒否する
```

その代わり、人間は以下を通じて classifier に判断材料を与える。

- 会話文脈
- `CLAUDE.md`
- `autoMode.environment`
- `autoMode.allow`
- `autoMode.soft_deny`
- `autoMode.hard_deny`
- `autoMode.classifyAllShell`

### permissions

人間が明示的に Claude Code の動作を制御する本命。

主に以下を設定する。

| 種類 | 意味 | 例 |
|---|---|---|
| `allow` | 自動で許可する | `terraform plan` |
| `ask` | 実行前に人間へ確認する | `terraform apply` |
| `deny` | 絶対に拒否する | `.env` 読み取り |

例:

```json
{
  "permissions": {
    "allow": [
      "Bash(terraform plan*)"
    ],
    "ask": [
      "Bash(terraform apply*)",
      "Bash(terraform destroy*)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)"
    ]
  }
}
```

勉強会では、以下の説明が分かりやすい。

```text
permissions は、Claude Code に対する明示的な交通ルール。
何を通すか、何を確認するか、何を絶対止めるかを人間が決める。
```

### hooks

Claude Code のツール実行前後などに、独自処理を差し込む仕組み。

permissions が静的なルールだとすると、hooks は部署ルールをコードで実装する仕組み。

代表的には `PreToolUse` と `PostToolUse` が使いやすい。

#### PreToolUse

実行前にチェックして止める。

例:

- `prod` / `production` / `本番` を含む操作を止める
- `terraform apply` を止める
- `kubectl delete` を止める
- Slack 投稿前に追加チェックする
- MCP の危険操作を止める

#### PostToolUse

実行後に記録する。

例:

- 実行コマンドをログに残す
- 作業ディレクトリを記録する
- 使用した MCP tool を記録する
- 成功 / 失敗を記録する

勉強会では、以下の説明が分かりやすい。

```text
hooks は会社独自の門番。
permissions だけでは表現しにくいルールや監査ログをコードで足せる。
```

## 評価順のイメージ

auto mode で Claude Code がツールを実行しようとしたときのイメージ。

```text
Claude Code がツールを実行しようとする
↓
permissions.deny に当たるか？
  → 当たるなら拒否
↓
permissions.ask に当たるか？
  → 当たるなら人間に確認
↓
permissions.allow に当たるか？
  → 条件によっては許可
↓
auto mode の classifier が安全性を判定
  → 危険そうならブロックまたは確認に戻す
  → 問題なさそうなら実行
↓
hooks が設定されていれば、PreToolUse / PostToolUse などで追加制御
```

実際の細かい順序や挙動は設定・ツール種別によって異なる可能性があるため、勉強会では以下の大枠で伝える。

```text
絶対に止めたいものは permissions.deny。
人間承認を挟みたいものは permissions.ask。
部署独自の判断や監査ログは hooks。
classifier はその後ろで auto mode を支える安全判定。
```

## 人間が制御する手段の優先度

### 1. permissions

最優先。

`.env`、`terraform apply`、MCP の delete / merge / post など、明確に制御したいものは permissions に書く。

例:

```text
.env は deny
terraform apply / destroy は ask
MCP の delete / merge は deny
Slack post は ask
```

### 2. hooks

permissions では表現しづらいルールを足す。

例:

```text
prod を含む操作は止める
実行したコマンドを監査ログに残す
特定の AWS account id を含む操作は止める
特定の MCP tool 実行時に追加チェックする
```

### 3. autoMode 設定

classifier に追加の判断材料を与える。

特に `autoMode.environment` は、自社の trusted boundary を説明するために使う。

例:

```json
{
  "autoMode": {
    "environment": [
      "$defaults",
      "Organization: Example Corp. Primary use: infrastructure automation",
      "Cloud provider(s): AWS",
      "Source control: GitHub org github.example.com/example-corp",
      "Protected IaC scopes: production AWS accounts and terraform/prod directories"
    ]
  }
}
```

### 4. CLAUDE.md

Claude と classifier への申し送り。

例:

```md
- production 環境への deploy は実行しない
- terraform apply / destroy は必ずユーザー確認を挟む
- .env や secrets 配下のファイルを読まない
- main / master への直接 push はしない
```

ただし、`CLAUDE.md` は強制力のある制御ではない。

強制したいものは permissions または hooks に寄せる。

### 5. 外部サービス側の権限設計

Claude Code の外側の制御。

実務ではかなり重要。

例:

- AWS IAM role を read-only にする
- 検証環境専用 role を使う
- GitHub token の権限を絞る
- main branch protection を有効にする
- Google Drive の対象フォルダを限定する
- Slack app の権限を絞る

## 比喩

### セキュリティゲート風

| 仕組み | 比喩 |
|---|---|
| permissions.deny | 絶対に開かない鍵付きドア |
| permissions.ask | 開ける前に人間へ確認するドア |
| permissions.allow | 通行許可証 |
| classifier | 通行人や荷物を見て止める警備員 |
| autoMode.environment | 警備員に渡す社内地図 |
| hooks | 会社独自の門番・監査カメラ |
| CLAUDE.md | 申し送りメモ |

### 勉強会タイトルに寄せるなら

| 仕組み | 比喩 |
|---|---|
| permissions | リード |
| hooks | 会社独自の門番 |
| classifier | 散歩中に危ない方向へ行かないか見る見張り |
| autoMode.environment | 散歩してよい範囲の地図 |
| CLAUDE.md | しつけメモ |
| 外部サービス側の権限 | そもそも入れない部屋を作る |

## 勉強会での説明例

### 30秒版

```text
classifier は Claude Code 側の内蔵安全判定です。
ただし、人間が直接きっちり制御する本命は permissions と hooks です。
絶対に止めたいものは deny、人間確認したいものは ask、部署独自ルールは hooks にします。
```

### 1分版

```text
auto mode では、通常の確認プロンプトを減らす代わりに classifier が裏側で安全性を判定します。
ただし classifier は万能ではなく、会社固有の事情を完全に知っているわけでもありません。

だから、人間側で permissions を使って、.env は deny、terraform apply は ask のように境界を決めます。
さらに、permissions だけでは足りない部署独自ルールや監査ログは hooks で補います。

つまり、classifier に任せるのではなく、permissions と hooks で柵を作ったうえで classifier に見張らせる、という考え方です。
```

## 1枚スライド案

```text
classifier / permissions / hooks の役割分担

permissions
- 人間が書く明示的ルール
- allow / ask / deny
- .env deny、terraform apply ask

hooks
- 会社独自の門番
- 実行前チェック、監査ログ
- prod 操作ブロックなど

classifier
- Claude Code 側の内蔵安全判定
- auto mode 中に危険そうな操作を判定
- ただし直接制御する本命ではない

結論:
classifier に任せるのではなく、
permissions と hooks で柵を作ってから auto mode を使う。
```

## 今回の勉強会での位置づけ

20分勉強会では、classifier を主役にしない。

主役は以下。

```text
1. auto mode は作業加速の本丸
2. でも放し飼いは怖い
3. permissions で .env deny / terraform apply ask
4. MCP は手足なので制御
5. classifier は裏側の安全判定として補足
```

classifier の説明は、permissions の基本説明の後に 30秒から1分だけ挟むのがよい。

```text
permissions が人間の明示的ルール。
auto mode では、その後ろで classifier が安全性を判定する。
ただし、絶対止めたいものは classifier 任せにせず permissions.deny に書く。
```
