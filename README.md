# GitHub Actions Collection

GitHub Actions で使える便利なアクションを提供するリポジトリです。

## 提供アクション一覧

| アクション名 | 説明 | ドキュメント |
|-------------|------|-------------|
| Check Repository Permission | リポジトリへの書き込み権限を確認 | [詳細](#check-repository-permission) |
| Download OpenH264 | プラットフォーム別に OpenH264 ライブラリをダウンロード | [詳細](#download-openh264) |
| Setup CUDA Toolkit | Linux と Windows 用の CUDA Toolkit をセットアップ | [詳細](#setup-cuda-toolkit) |

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

### Download OpenH264

指定したプラットフォーム用の OpenH264 ライブラリをダウンロードするアクションです。
複数のプラットフォーム（Linux、macOS、Windows）に対応しており、CI/CD 環境で OpenH264 を必要とする場合に便利です。

このアクションは GitHub CLI (`gh`) を使用して cisco/openh264 の公式リリースから動的に URL を取得するため、ライブラリのバージョン番号（.so.8 など）が変更されても自動的に対応します。
また、ダウンロード後は MD5 チェックサムによる整合性検証も自動的に行います。

オプションでキャッシュ機能を有効にすることで、同じバージョンのライブラリを再ダウンロードすることなく高速にビルドを実行できます。

#### 基本的な使い方

```yaml
- uses: shiguredo/github-actions/.github/actions/download-openh264@main
  id: openh264
  with:
    platform_name: ubuntu-24.04_x86_64
    openh264_version: 2.6.0
```

#### 入力パラメータ

| 名前 | 説明 | 必須 | デフォルト |
|------|------|------|------------|
| `platform_name` | プラットフォーム名（例: `ubuntu-24.04_x86_64`, `macos_arm64`, `windows_x86_64`） | ✓ | - |
| `openh264_version` | OpenH264のバージョン（例: `2.6.0`） | ✓ | - |
| `use-cache` | ダウンロードしたライブラリをキャッシュするか（`true`/`false`） | - | `false` |

#### 出力

| 名前 | 説明 | 例 |
|------|------|-----|
| `openh264_path` | ダウンロードした OpenH264 ライブラリのパス | `/path/to/libopenh264.so` |
| `cache-hit` | キャッシュがヒットしたか | `true` または `false` |

#### 対応プラットフォーム

| プラットフォーム | 値 | ライブラリファイル |
|-----------------|-----|-------------------|
| Ubuntu (x86_64) | `ubuntu-*_x86_64` | `libopenh264.so` |
| Ubuntu (ARM64) | `ubuntu-*_armv8` | `libopenh264.so` |
| macOS (ARM64) | `macos_arm64` or `macos-*_arm64` | `libopenh264.dylib` |
| Windows (x86_64) | `windows_x86_64` or `windows-*_x86_64` | `libopenh264.dll` |

#### 使用例

<details>
<summary>Linux での OpenH264 ダウンロードと使用</summary>

```yaml
name: Build with OpenH264

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      
      - uses: shiguredo/github-actions/.github/actions/download-openh264@main
        id: openh264
        with:
          platform_name: ubuntu-24.04_x86_64
          openh264_version: 2.6.0
      
      - name: Build with OpenH264
        run: |
          export LD_LIBRARY_PATH="${{ steps.openh264.outputs.openh264_path }}:$LD_LIBRARY_PATH"
          make build
```

</details>

<details>
<summary>マルチプラットフォーム対応ビルド</summary>

```yaml
name: Multi-platform Build

on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: ubuntu-24.04
            platform: ubuntu-24.04_x86_64
          - os: macos-latest
            platform: macos_arm64
          - os: windows-latest
            platform: windows_x86_64
    
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      
      - uses: shiguredo/github-actions/.github/actions/download-openh264@main
        id: openh264
        with:
          platform_name: ${{ matrix.platform }}
          openh264_version: 2.6.0
      
      - name: Use OpenH264
        run: |
          echo "OpenH264 library is at: ${{ steps.openh264.outputs.openh264_path }}"
```

</details>

<details>
<summary>キャッシュを有効にした高速ビルド</summary>

```yaml
name: Cached Build

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      
      - uses: shiguredo/github-actions/.github/actions/download-openh264@main
        id: openh264
        with:
          platform_name: ubuntu-24.04_x86_64
          openh264_version: 2.6.0
          use-cache: 'true'
      
      - name: Show cache status
        run: |
          if [[ "${{ steps.openh264.outputs.cache-hit }}" == "true" ]]; then
            echo "OpenH264 was loaded from cache"
          else
            echo "OpenH264 was downloaded and cached"
          fi
      
      - name: Build with OpenH264
        run: |
          export LD_LIBRARY_PATH="${{ steps.openh264.outputs.openh264_path }}:$LD_LIBRARY_PATH"
          make build
```

</details>

### Setup CUDA Toolkit

Linux (Ubuntu 22.04/24.04) と Windows 用の CUDA Toolkit をセットアップするアクションです。
CI/CD 環境で CUDA を必要とするビルドやテストを行う場合に便利です。

Linux では NVIDIA の公式リポジトリから CUDA Toolkit をインストールし、Windows ではキャッシュ機能を提供します。

**注意:** このアクションは x86_64 アーキテクチャのみをサポートしています。ARM アーキテクチャでは動作しません。

#### 基本的な使い方

```yaml
- uses: shiguredo/github-actions/.github/actions/setup-cuda-toolkit@main
  id: cuda
  with:
    cuda_version: 12.9.1
    platform: ubuntu-24.04
```

#### 入力パラメータ

| 名前 | 説明 | 必須 | デフォルト |
|------|------|------|------------|
| `cuda_version` | CUDA バージョン（例: `12.9.1`） | ✓ | - |
| `platform` | プラットフォーム（`ubuntu-22.04`, `ubuntu-24.04`, `windows-2022`, `windows-2025`） | ✓ | - |
| `use-cache` | CUDA インストールをキャッシュするか（`true`/`false`） | - | `true` |

#### 出力

| 名前 | 説明 | 例 |
|------|------|-----|
| `cuda_path` | CUDA インストールパス | `/usr/local/cuda` |
| `cache-hit` | キャッシュがヒットしたか（Windows のみ） | `true` または `false` |

#### 利用可能な CUDA バージョン

2025 年 10 月現時点で利用可能な主な CUDA バージョン:

**Ubuntu 22.04 / 24.04 / Windows:**

- CUDA 12.x: `12.5.1`, `12.6.0`, `12.6.1`, `12.6.2`, `12.6.3`, `12.8.0`, `12.8.1`, `12.9.0`, `12.9.1`
- CUDA 13.x: `13.0.0`, `13.0.1`, `13.0.2`

**注意:**
- Ubuntu 24.04 では CUDA 12.5.1 以降が利用可能です
- Ubuntu ではバージョンに自動的に `-1` が付加されてインストールされます（例: `12.9.1` → `cuda-toolkit-12=12.9.1-1`）

最新の利用可能なバージョンについては、NVIDIA の公式サイトを確認してください:

- Ubuntu 22.04: <https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/>
- Ubuntu 24.04: <https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/>
- Windows: <https://developer.download.nvidia.com/compute/cuda/>

#### 使用例

<details>
<summary>Ubuntu 24.04 での CUDA セットアップとビルド</summary>

```yaml
name: Build with CUDA

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - uses: shiguredo/github-actions/.github/actions/setup-cuda-toolkit@main
        id: cuda
        with:
          cuda_version: 12.9.1
          platform: ubuntu-24.04

      - name: Verify CUDA installation
        run: |
          nvcc --version
          echo "CUDA path: ${{ steps.cuda.outputs.cuda_path }}"

      - name: Build with CUDA
        run: |
          export PATH=/usr/local/cuda/bin:$PATH
          export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
          make build
```

</details>

<details>
<summary>マルチバージョン CUDA テスト</summary>

```yaml
name: Multi-CUDA Build

on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: ubuntu-24.04
            platform: ubuntu-24.04
            cuda_version: 12.9.1
          - os: ubuntu-24.04
            platform: ubuntu-24.04
            cuda_version: 13.0.2
          - os: ubuntu-22.04
            platform: ubuntu-22.04
            cuda_version: 12.8.1

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: shiguredo/github-actions/.github/actions/setup-cuda-toolkit@main
        with:
          cuda_version: ${{ matrix.cuda_version }}
          platform: ${{ matrix.platform }}

      - name: Build and test
        run: |
          export PATH=/usr/local/cuda/bin:$PATH
          export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
          make test
```

</details>

<details>
<summary>Windows でのキャッシュを利用した CUDA セットアップ</summary>

```yaml
name: Windows CUDA Build

on: [push, pull_request]

jobs:
  build:
    runs-on: windows-2025
    steps:
      - uses: actions/checkout@v4

      - uses: shiguredo/github-actions/.github/actions/setup-cuda-toolkit@main
        id: cuda
        with:
          cuda_version: 12.9.1
          platform: windows-2025
          use-cache: 'true'

      - name: Show cache status
        shell: pwsh
        run: |
          Write-Host "CUDA path: ${{ steps.cuda.outputs.cuda_path }}"
          Write-Host "Cache hit: ${{ steps.cuda.outputs.cache-hit }}"

      - name: Verify CUDA installation
        shell: pwsh
        run: |
          $env:PATH = "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.9.1\bin;$env:PATH"
          nvcc --version

      - name: Build
        shell: pwsh
        run: |
          $env:PATH = "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.9.1\bin;$env:PATH"
          # ビルドコマンド
          make build
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
