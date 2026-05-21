---
version: "1.1.0"
---

# PowerShell 実行環境ルール

本プロジェクトの開発・運用は Windows + PowerShell (pwsh) を前提とする。
チーム共通で利用可能なツールは PowerShell と Python 3。`jq` は未導入。

## API レスポンスの文字化けを見たら表示側を疑う

バックエンド/フロントエンドの API レスポンスは **すべて UTF-8 で返却される** 前提が成立している場合、
コンソール上で日本語が化けて見えたら **サーバー側のバグではなく PowerShell コンソールの表示レイヤー（cp932）による化け** である可能性が高い。
まず次のいずれかの方法で UTF-8 表示にしてから再確認すること:

```powershell
# セッションのコンソール出力を UTF-8 に切替
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

# または、レスポンスをファイル保存して UTF-8 対応エディタ / Read ツールで読む
curl.exe -sS -H "X-API-Key: $apikey" "https://..." -o $env:TEMP\resp.json

# または、Python3 の json.tool で整形（--no-ensure-ascii で日本語をそのまま出力）
curl.exe -sS -H "X-API-Key: $apikey" "https://..." -o $env:TEMP\resp.json
python3 -m json.tool --no-ensure-ascii $env:TEMP\resp.json
```

生バイト列を検証したい場合は `od -c` や Python3 の `.encode('utf-8').hex()` を使う。

## `/dev/null` は使えない

Bash の `cmd > /dev/null` 相当は PowerShell では `| Out-Null` または `$null` への代入:

```powershell
# 出力を捨てる
cmd | Out-Null
$null = cmd

# AWS CLI など「出力ファイルパス」を要求するコマンドは $env:TEMP 配下を使う
aws lambda invoke --function-name xxx $env:TEMP\lambda_out.json
```

## `gh`/`aws` の出力を `python3 -c` 等にパイプしない

PowerShell のパイプは文字列処理が独自で、Python の f-string 内の `[`・`\"`・`#` などが途中で壊れることがある。また `.exe` 系コマンドの出力完了前にパイプが閉じられ「パイプを閉じています」エラーになる場合がある。

```powershell
# NG（壊れる可能性）
gh api "repos/{owner}/{repo}/issues" | python3 -c "import sys,json; ..."

# OK: --jq を使う（gh 組み込み）
gh api "repos/{owner}/{repo}/issues" --jq ".[].title"

# OK: 一旦変数に受けて ConvertFrom-Json で処理する
$json = gh api "repos/{owner}/{repo}/issues"
$json | ConvertFrom-Json | ForEach-Object { ... }

# OK: 一旦ファイルに保存してから python3 で読み込む
gh api "repos/{owner}/{repo}/issues" | Out-File $env:TEMP\issues.json -Encoding utf8
python3 -c "import json; d=json.load(open(r'$env:TEMP\issues.json', encoding='utf-8')); ..."
```

## Git Bash での MSYS パス変換

Git Bash 経由でコマンドを実行すると、`/aws/...`・`/v1/...` のような先頭スラッシュ引数が自動的に Windows パス（`C:/Program Files/Git/aws/...`）に変換されてしまう:

```bash
MSYS_NO_PATHCONV=1 aws ssm send-command --document-name "AWS-RunShellScript" ...
```

PowerShell から直接 `aws`/`gh` を呼ぶ場合は該当しない。

## Unix 互換エイリアスは使用禁止・PowerShell ネイティブコマンドレットを使う

PowerShell には `ls`・`cat`・`cp`・`mv`・`rm`・`mkdir`・`echo` 等の Unix 互換エイリアスが存在するが、**これらを使用してはいけない**。
PowerShell ネイティブのコマンドレットを使うこと。

| Unix 互換エイリアス | 使用禁止 | PowerShell ネイティブ（使用必須） |
|---|---|---|
| `ls` | 禁止 | `Get-ChildItem` |
| `cat` | 禁止 | `Get-Content` |
| `cp` | 禁止 | `Copy-Item` |
| `mv` | 禁止 | `Move-Item` |
| `rm` | 禁止 | `Remove-Item` |
| `mkdir` | 禁止 | `New-Item -ItemType Directory` |
| `echo` | 禁止 | `Write-Output`（または文字列を直接出力） |
| `pwd` | 禁止 | `Get-Location` |

**例外**: SSM Run Command の `--parameters 'commands=[...]'` のように Linux EC2 上で実行するコマンド文字列の中では Bash コマンドを使う（サーバー側は Linux）。

## `curl` は `.exe` を明示

PowerShell の `curl` は `Invoke-WebRequest` のエイリアスで挙動が異なる。HTTP クライアントとして `curl` を使うなら `curl.exe` と明示する:

```powershell
# NG: Invoke-WebRequest が呼ばれる
curl https://example.com/api

# OK: ネイティブの curl を使用
curl.exe -sS https://example.com/api
```

## Python スクリプトの UTF-8 出力

Windows の Python はデフォルトでシステムコードページ（cp932 / Shift_JIS）を stdout / stderr に使う。`Start-Process -RedirectStandardOutput` や `| Out-File` でログをファイルに書く場合、日本語が文字化けする。

### スクリプト側の対処（必須）

全 Python スクリプトの冒頭（import 直後）に以下を入れる:

```python
import sys
sys.stdout.reconfigure(encoding="utf-8")
sys.stderr.reconfigure(encoding="utf-8")
```

### 呼び出し側の対処（補助）

スクリプト側を修正できない場合は、環境変数で強制する:

```powershell
$env:PYTHONIOENCODING = "utf-8"
python3 script.py
```

## 関連

- `gh-commands.md`: gh コマンド運用ルール
- `known-pitfalls.md`: その他の既知の落とし穴
