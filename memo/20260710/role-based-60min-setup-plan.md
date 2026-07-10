# 参加者別ゴールに分ける 60 分セットアップ会案

## 前提

セットアップ会の参加者は 4 名。

参加者ごとに立場、Claude Code の利用状況、求めているゴールが違う。

| 参加者 | 立場 / 状態 | 関心 |
|---|---|---|
| A さん | マネージャー | Backlog MCP / Slack MCP が有効 |
| B さん | マネージャー | Backlog MCP / Slack MCP が有効 |
| C さん | 一般社員。既に Claude Code を個人利用していた | どこまで設定できているか不明 |
| D さん | 派遣社員 | Backlog MCP / Slack MCP より settings.json などの制御を重視 |

この内訳だと、全員に同じゴールを設定するのは合わない。

A さん、B さんは MCP 活用が主目的になりやすい。

C さんは個人利用済みなので、組織設定との整合確認が重要。

D さんは権限や制御の理解が重要。

## 結論

60 分の会を、Backlog MCP 会または settings.json 会のどちらか一方として設計しない。

会のテーマは以下にする。

```text
Claude Code を各自の業務権限に合わせて安全に使い始める会
```

最初の 15 分だけ共通確認を行い、その後は参加者別のゴールに分岐する。

```text
共通確認 15 分
個別ゴール 35 分
共有と次アクション 10 分
```

## なぜ分岐させるか

60 分で以下を全員に同じ深さでやるのは難しい。

- Backlog MCP
- Slack MCP
- settings.json
- permissions
- managed settings
- 既存個人設定の棚卸し

しかも参加者ごとに必要なものが違う。

そのため、全員を同じ一本道で進めると以下が起きる。

- A さん、B さんには permissions 深掘りが重すぎる
- D さんには Backlog MCP / Slack MCP 活用が主目的とズレる
- C さんには初期インストール説明が退屈になる可能性がある
- 主催者が全員の詰まりを同時に吸収できない

よって、会の設計は「全員同じゴール」ではなく「共通の安全確認 + 個別ゴール」にする。

## 共通ゴール

全員に共通するゴールは、以下だけに絞る。

```text
Claude Code が組織の管理設定を読み込んでいるか確認する。
MCP や settings.json は、立場と権限に合わせて扱うものだと理解する。
```

共通でやる確認。

```bash
claude --version
claude doctor
claude mcp list
```

Claude Code 内で確認するもの。

```text
/status
/mcp
```

見るポイント。

- Claude Code が起動できるか
- managed settings が効いているか
- Backlog MCP / Slack MCP が見えているか
- 個人設定や project 設定が入っているか
- permissions が組織管理されているか

## 参加者別ゴール

### A さん、B さん

マネージャー。

Backlog MCP / Slack MCP が有効。

当日のゴール。

```text
MCP を使って、業務情報を read できることを確認する。
```

やること。

- Backlog の担当課題、確認中課題、期限が近い課題を取得する
- 課題を 1 件要約する
- 課題一覧から今日見るべきものを抽出する
- Slack は read 系だけ確認する
- post / update / delete は当日は実行しない

依頼例。

```text
Backlog MCP を使って、私が担当している未完了課題を一覧にして
```

```text
Backlog MCP を使って、期限が近い課題を 5 件出して、優先度順に整理して
```

```text
Slack MCP を使って、今日の関連チャンネルの未読内容を要約して
```

注意点。

```text
初回は read 系だけ。
投稿、課題更新、ステータス変更、削除は扱わない。
```

### C さん

一般社員。

既に Claude Code を個人で使っていた。

どこまで設定できているか不明。

当日のゴール。

```text
個人利用の設定と、組織管理の設定が競合していないか棚卸しする。
```

やること。

- `claude doctor` を確認する
- `/status` で managed settings の有無を見る
- `claude mcp list` で個人で追加した MCP を確認する
- `~/.claude/settings.json` に個人設定があるか確認する
- `.claude/settings.json` や `.claude/settings.local.json` があるか確認する
- 個人で入れた permissions / hooks / MCP が組織利用として危なくないか確認する

確認したい観点。

- 個人で強すぎる MCP を追加していないか
- 組織の managed settings を上書きしようとしていないか
- `.env` や secrets を読める状態になっていないか
- auto mode を強くしすぎていないか
- write 系 MCP を無条件許可していないか

依頼例。

```text
今の Claude Code の設定状況を棚卸ししたい。
managed settings、個人 settings、MCP、permissions の観点で確認項目を出して。
```

```text
Claude Code の現在の設定で、組織利用として危なそうな点がないか確認して。
変更前に必ず差分を説明して。
```

### D さん

派遣社員。

Backlog MCP / Slack MCP よりも、settings.json などで制御をしっかり設定したい。

当日のゴール。

```text
settings.json / permissions / managed settings の考え方を理解し、安全な初期設定を確認する。
```

やること。

- settings.json の場所を確認する
- managed settings が効いているか確認する
- permissions の `allow` / `ask` / `deny` を理解する
- `.env` / `secrets/**` / `~/.aws/**` / `~/.ssh/**` を読ませない考え方を確認する
- MCP の read / write / delete を分ける考え方を確認する
- 自分で追加できる設定と、組織管理で決まる設定の境界を理解する

依頼例。

```text
Claude Code の settings.json と permissions の考え方を説明して。
特に allow / ask / deny の使い分けを、業務利用の観点で整理して。
```

```text
初回利用として安全寄りの permissions 設定案を作って。
.env、secrets、~/.aws、~/.ssh は読ませない。
MCP は read 系だけ許可し、write 系は確認、delete 系は拒否する方針にして。
```

注意点。

`allowManagedPermissionRulesOnly` が managed settings で有効な場合、ユーザーや project 側で permissions を追加できない。

その場合は、当日に設定を入れるのではなく、どの制御が組織側で効いているかを確認する会にする。

## 60 分の進行案

| 時間 | 内容 |
|---|---|
| 0:00 - 0:05 | 今日の方針説明 |
| 0:05 - 0:15 | 全員で現在地確認 |
| 0:15 - 0:35 | 参加者別ルートで作業 |
| 0:35 - 0:50 | 各自のゴール確認 |
| 0:50 - 1:00 | 共有、未完了事項、次アクション |

## 0:00 - 0:05 今日の方針説明

最初に、全員同じことをやる会ではないと明示する。

話すこと。

```text
今日は全員で同じ手順を進める会ではありません。
立場と現在の設定状況が違うので、最初だけ共通確認をして、
その後は各自のゴールに分けます。
```

共通メッセージ。

```text
同じ Claude Code でも、立場によって初期設定の正解は違います。
マネージャーは MCP 活用。
個人利用済みの人は設定棚卸し。
制御を重視する人は settings.json / permissions の確認。
```

## 0:05 - 0:15 全員で現在地確認

全員に以下を実行してもらう。

```bash
claude --version
claude doctor
claude mcp list
```

Claude Code 内で確認。

```text
/status
/mcp
```

この時点で見ること。

- A さん、B さんは Backlog MCP / Slack MCP が見えるか
- C さんは個人設定や個人 MCP が残っていないか
- D さんは managed settings / permissions の状態が確認できるか

## 0:15 - 0:35 参加者別ルートで作業

### A さん、B さん

MCP 活用ルート。

```text
Backlog MCP / Slack MCP の read 系だけを使って、業務情報を取れるか確認する。
```

### C さん

設定棚卸しルート。

```text
既存の個人設定と組織設定の差分を確認する。
```

### D さん

制御確認ルート。

```text
settings.json / permissions / managed settings の考え方を確認する。
```

## 0:35 - 0:50 各自のゴール確認

各参加者ごとに、以下を確認する。

### A さん、B さん

- Backlog MCP が見えているか
- 課題を read できたか
- Slack MCP は read 系だけ確認できたか
- write 系操作をしない方針を理解したか

### C さん

- 個人設定がどこにあるか分かったか
- managed settings が効いているか分かったか
- 危なそうな個人設定があるか分かったか
- 次に消す / 直す / 管理者に確認するものが明確になったか

### D さん

- settings.json の場所が分かったか
- managed settings と個人設定の違いが分かったか
- allow / ask / deny の考え方が分かったか
- 自分で設定すべき範囲と、組織側で決める範囲が分かったか

## 0:50 - 1:00 共有、未完了事項、次アクション

最後に全員で共有する。

共有する項目。

- 今日できたこと
- 詰まったこと
- 次に確認する人
- 管理者に確認すること
- 次回扱うテーマ

次回テーマ候補。

- Backlog MCP の write 系操作を ask で扱う
- Slack MCP の投稿を安全に扱う
- permissions の標準設定を作る
- managed settings の中身を確認する
- hooks で部署ルールを入れる

## 当日の成功条件

全員が同じ完成状態にならなくてよい。

成功条件は参加者ごとに分ける。

| 参加者 | 成功条件 |
|---|---|
| A さん、B さん | Backlog / Slack MCP の read 系活用イメージが持てる |
| C さん | 個人設定と組織設定の棚卸しポイントが分かる |
| D さん | settings.json / permissions / managed settings の制御方針が分かる |

共通の成功条件。

```text
Claude Code は、立場と権限に合わせて設定するものだと理解する。
MCP は便利機能ではなく、外部サービス操作権限として扱う。
write 系操作は、初回セットアップ会では扱わない。
```

## 主催者の判断

この会では、Backlog MCP と permissions のどちらか一方に全員を寄せない。

ただし時間が 60 分なので、説明は共通化しすぎない。

主催者は以下の役割に徹する。

- 最初に全員の現在地を揃える
- 参加者別にゴールを割り当てる
- write 系操作を止める
- 詰まり箇所を次アクションに落とす

## まとめ

この 4 人の内訳では、全員同じセットアップ会にしない。

最初の 15 分だけ共通確認し、その後は参加者別に分岐する。

```text
A さん、B さん: MCP 活用
C さん: 個人設定と組織設定の棚卸し
D さん: settings.json / permissions / managed settings の制御確認
```

これが 60 分で一番破綻しにくい。
