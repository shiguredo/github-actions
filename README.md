# GitHub Actions Collection

GitHub Actions で使える便利なアクションを提供するリポジトリです。

## 提供アクション一覧

| アクション名 | 説明 | ドキュメント |
|-------------|------|-------------|
| Check Repository Permission | リポジトリへの書き込み権限を確認 | [詳細](#check-repository-permission) |

## アクション詳細

### Check Repository Permission

指定したユーザーがリポジトリへの書き込み権限を持っているかを確認するアクションです。外部コントリビューターやメンバーの権限レベルを判定し、権限に応じた処理を行いたい場合に便利です。

このアクションは GitHub の認証トークン（`github.token`）を自動的に使用して権限を確認します。

#### 基本的な使い方

```yaml
- uses: shiguredo/github-actions/.github/actions/check-write-permission@main
  id: check
  with:
    username: ${{ github.actor }}
```

#### 入力パラメータ

| 名前 | 説明 | 必須 | デフォルト |
|------|------|------|------------|
| `username` | 権限を確認する GitHub ユーザー名 | ✓ | - |

#### 出力

| 名前 | 説明 | 値 |
|------|------|-----|
| `has-permission` | 書き込み権限の有無 | `true` または `false` |
| `permission-level` | 権限レベル | `admin`, `write`, `maintain`, `read`, `none` |

#### 使用例

<details>
<summary>外部コントリビューターの PR に自動ラベルを付ける</summary>

```yaml
name: Auto Label External PRs

on:
  pull_request:
    types: [opened]

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: shiguredo/github-actions/.github/actions/check-write-permission@main
        id: permission
        with:
          username: ${{ github.event.pull_request.user.login }}
      
      - name: Add external label
        if: steps.permission.outputs.has-permission == 'false'
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              labels: ['external-contribution']
            })
```

</details>

<details>
<summary>権限に基づいて異なる CI を実行</summary>

```yaml
name: Conditional CI

on:
  pull_request:

jobs:
  check-permission:
    runs-on: ubuntu-latest
    outputs:
      has-write: ${{ steps.check.outputs.has-permission }}
    steps:
      - uses: shiguredo/github-actions/.github/actions/check-write-permission@main
        id: check
        with:
          username: ${{ github.actor }}
  
  internal-ci:
    needs: check-permission
    if: needs.check-permission.outputs.has-write == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running internal CI with secrets"
  
  external-ci:
    needs: check-permission
    if: needs.check-permission.outputs.has-write == 'false'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running external CI without secrets"
```

</details>

## 必要な権限

このアクションを使用するには、GitHub トークンに以下の権限が必要です：

- `contents: read` - リポジトリの情報を読み取るため
- `metadata: read` - リポジトリのメタデータを読み取るため（デフォルトで付与）

## ライセンス

Apache License 2.0

```text
Copyright 2025-2025, Shiguredo Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
