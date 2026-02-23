---
name: github-actions-slack-notify
description: shiguredo/github-actions リポジトリの Slack Notify Composite Action の開発・修正・レビュー。入力パラメータ、notify_mode、Fixed 通知、色判定、フィールド制御、ペイロード構築、テストワークフローに関する質問で使用。
---

# Slack Notify Composite Action

shiguredo/github-actions リポジトリで提供される Slack 通知用 Composite Action。

## リポジトリ構成

- アクション定義: `.github/actions/slack-notify/action.yml`
- テストワークフロー: `.github/workflows/test-slack-notify.yml`
- rtCamp/action-slack-notify の互換実装 (Docker レス)
- 元リポジトリ (参考): `/tmp/action-slack-notify/`

## 入力パラメータ

| 名前 | 必須 | デフォルト | 説明 |
|------|------|-----------|------|
| `status` | 必須 | - | ジョブステータス (`job.status` を渡す) |
| `slack_webhook` | 必須 | - | Slack Incoming Webhook URL |
| `slack_channel` | - | `''` | チャネル上書き |
| `slack_title` | - | `''` | メッセージタイトル (空なら自動生成: "ステータス: ワークフロー名") |
| `slack_message` | - | `''` | メッセージ本文 (空なら最新コミットメッセージ) |
| `slack_color` | - | `''` | 色の手動指定 (指定時は自動判定を上書き) |
| `slack_username` | - | `GitHub Actions` | ボット名 |
| `slack_icon_emoji_success` | - | `:yatta:` | Success 時のボットアバター |
| `slack_icon_emoji_failure` | - | `:chiin:` | Failure 時のボットアバター |
| `slack_icon_emoji_fixed` | - | `:fixed:` | Fixed 時のボットアバター |
| `slack_footer` | - | `Powered by shiguredo/github-actions` | フッター |
| `msg_minimal` | - | `''` | `true` で最小表示、カンマ区切りで個別指定 (ref,event,actions_url,commit) |
| `notify_mode` | - | `failure_and_fixed` | 通知モード |

## notify_mode

| モード | 動作 |
|--------|------|
| `all` | 全ステータスで通知 |
| `failure_and_fixed` | failure と fixed のみ |
| `failure_only` | failure のみ |
| `success_only` | success のみ |

## 処理フロー

1. ステータスを小文字に正規化
2. `gh run list` で前回のワークフロー実行結果を取得 (Fixed 判定用)
3. `notify_mode` に基づく送信スキップ判定
4. 色の決定 (手動指定 > 自動判定)
5. ステータステキスト決定 (Fixed / Success / Failure / Cancelled)
6. タイトル・メッセージ生成
7. `msg_minimal` に応じたフィールド配列構築
8. `jq -n` で安全に JSON ペイロード生成
9. `curl` で Slack Webhook に POST
10. HTTP ステータスコード確認

## Fixed 通知

- 条件: 前回 failure かつ 今回 success
- 色: `#2196F3` (青)
- ステータステキスト: `Fixed`
- `actions: read` 権限が必要 (`gh run list` のため)

## 色の自動判定

| 条件 | 色 |
|------|-----|
| `slack_color` 指定済み | 指定値 |
| success + 前回 failure (Fixed) | `#2196F3` |
| success | `good` |
| failure | `danger` |
| その他 | `#808080` |

## フィールド制御 (msg_minimal)

- 空: 全フィールド (Ref, Event, Actions URL, Commit, Status)
- `true`: Status のみ
- カンマ区切り (例: `ref,commit`): 指定フィールド + Status

## テストワークフロー

`test-slack-notify.yml` は `workflow_dispatch` で手動実行する。

入力:
- `force_failure` (boolean): 意図的にジョブを失敗させる
- `notify_mode` (choice): failure_and_fixed / all / failure_only / success_only
- `slack_channel` (string): 通知先チャネル

必要な権限: `actions: read`
runs-on: `ubuntu-slim`

## rtCamp/action-slack-notify との差異

- Docker レス (Composite Action) のため起動が高速
- `status` 入力を追加 (rtCamp は環境変数 `SLACK_COLOR` で `${{ job.status }}` を渡す)
- `notify_mode` を追加 (all / failure_and_fixed / failure_only / success_only)
- Fixed 通知機能を追加 (前回 failure → 今回 success の自動検知)
- 環境変数ではなく `with` で入力を渡す
- `jq` で安全に JSON を構築 (rtCamp は Go / シェル文字列連結)

## 実装上の注意

- 環境変数経由でパラメータを受け取る (`env:` で `INPUT_*` にマッピング)
- `GH_TOKEN: ${{ github.token }}` が必要 (`gh run list` のため)
- `GITHUB_*_VAL` で GitHub コンテキスト値を渡す (シェル内での `${{ }}` 展開を避ける)
- `set -euo pipefail` で厳密なエラーハンドリング
- `jq -n` による安全な JSON 生成 (シェルインジェクション防止)
- HTTP ステータスコードが 200-299 以外の場合は `::warning::` で通知
