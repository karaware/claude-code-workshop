# claude-code-workshop

Claude Code の社内勉強会用リポジトリです。

## 目的

部署内で Claude Code の活用ノウハウを共有し、手順書作成・構成図作成・ナレッジ蓄積・安全な自走運用などを段階的に学ぶための資料置き場です。

## 勉強会テーマ

### 1. Claude Code で手順書自動作成

Claude Code を使って作業内容やコマンド実行結果から手順書を作成するテーマ。

### 2. Claude Code で構成図自動作成・ノウハウ蓄積

Claude Code を使った構成図作成と、自作ノウハウ蓄積ツールの紹介。

### 3. AI を飼い慣らせ

Claude Code を制御して安全に自走させるための勉強会案。

対象にする主な不安:

- AWS アカウントなどの共有リモート環境を壊すのが怖い
- Google Drive / GitHub / Slack などで誤操作するのが怖い
- 機密情報の取り扱いミスが怖い
- auto mode で自走させたいが、任せきるのは怖い

資料:

- [AI を飼い慣らせ - Claude Code を制御して安全に自走させる](./ai-tame-claude-code-safety.md)

## 今後扱いたい内容

- `permissions` による許可・確認・拒否の設計
- `permission mode` と auto mode の安全な使い方
- `hooks` による危険操作ブロックと監査ログ
- MCP 経由の外部サービス操作の制御
- AWS / Google Drive / GitHub / Slack など共有環境での事故防止
- sandbox の位置づけ
- managed settings の概要

## 基本方針

Claude Code を安全に使うとは、Claude を信じることではありません。

Claude が触れる権限・道具・環境を先に設計して、その範囲内で自走させることを目指します。
