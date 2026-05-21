---
version: "1.1.0"
---

# 破壊的パス削除の安全ガード

## 背景

**根本原因: Claude Code の PowerShell ツールは変数をツール呼び出し間で保持しない。**

パスを算出する PowerShell 呼び出しと `Remove-Item` を実行する PowerShell 呼び出しを分けると、変数が `$null` にリセットされる。`$null -replace '[^a-zA-Z0-9-]', '-'` は空文字列を返すため、`Join-Path` が親ディレクトリそのものを指し、`.claude/projects/` 配下が全消失する。

この事故パターンは Claude Code の PowerShell ツールを使うすべてのプロジェクトで発生しうる。

## MUST

### 1. Remove-Item のパス引数に変数を使う場合は null/空文字チェック必須

```powershell
# NG: ガードなし
Remove-Item -LiteralPath $targetDir -Recurse -Force

# OK: ガードあり
if (-not [string]::IsNullOrWhiteSpace($targetDir)) {
    Remove-Item -LiteralPath $targetDir -Recurse -Force
}
```

### 2. パス変数の設定から Remove-Item まで同一 PowerShell コマンドブロック内に収める

**これが最も重要なルール。** パスを算出する処理と削除する処理を別々の PowerShell 呼び出しに分けると、変数が `$null` になり誤削除の原因となる。

```powershell
# NG: 2回に分けて呼び出す（変数が消える）
# 呼び出し1: $targetDir = Join-Path ...
# 呼び出し2: Remove-Item $targetDir  ← $null になっている

# OK: 1回の呼び出しで完結させる
$targetDir = Join-Path $baseDir $subDir
if (-not [string]::IsNullOrWhiteSpace($targetDir)) {
    Remove-Item -LiteralPath $targetDir -Recurse -Force
}
```

### 3. 保護対象ディレクトリの削除はユーザー確認必須

以下のディレクトリ配下を `Remove-Item` する場合、削除対象パスの全文を表示してユーザー承認を得てから実行する:

- `$env:USERPROFILE\.claude\projects\`
- `$env:USERPROFILE\.claude\memory\`
- `$env:USERPROFILE\.claude\` 直下のディレクトリ

### 4. パスの深さチェック

算出されたパスのセグメント数が期待より少ない場合は中止する。
例: `.claude/projects/<name>` の `<name>` 部分が空だと `.claude/projects/` 自体を指してしまう。

```powershell
# 期待: $claudeProjName は "C--Users-username-..." のような長い文字列（数十文字以上）
if ($claudeProjName.Length -lt 10) {
    Write-Output "ERROR: claudeProjName が異常に短い ('$claudeProjName')。削除を中止します。"
    return  # ガード節: このチェック以降の削除処理を実行しない
}
```

閾値 `10` の根拠: 正常なパス変換結果は `C--Users-<username>-...` で最低でも20文字以上。10文字未満は明らかに異常。

**注意:** このチェックはガード節として使用する。`if` ブロック後に削除処理が続く構造にしないこと。`return` / `throw` / `exit` で明示的に処理を停止する。

## 適用範囲

全スキル・全手動操作に適用する。特に start-task の手順 0c（ワークツリー削除後の `.claude/projects/` クリーンアップ）で必須。
