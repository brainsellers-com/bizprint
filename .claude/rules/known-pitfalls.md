---
version: "2.0.0"
has_placeholders: true
description: "既知の落とし穴（Claude Code 汎用 + PJ固有追記用）"
---

# 既知の落とし穴

開発中に発見された、繰り返しハマりやすい問題のまとめ。

## エージェント調査結果の裏取り

Explore エージェントがリソースの有無を誤報告した事例がある（「存在しない」と報告したが実際は存在）。重要な判断の前は grep 等で自分でも確認する。

## Monitor ツールのコマンドは bash で実行される

Monitor ツールの `command` は `/usr/bin/bash` で実行される。PowerShell 構文をそのまま書くと `syntax error` で exit 2 になる。

Monitor コマンドには必ず bash 構文を使うか、`pwsh -NoProfile -Command "..."` でラップする:

```bash
# OK: bash 構文
while true; do
  STATUS=$(gh api "..." 2>/dev/null | python3 -c "..." 2>/dev/null || echo "api_error")
  case $STATUS in success) exit 0;; esac
  sleep 30
done
```

pwsh ラップの場合も監視ロジック（ステータス取得・判定・exit）を省略しないこと。
実装例は `bs-cc-plugins` プラグインの `_shared/ci-monitor.md` テンプレートを bash 構文で使用すること（定義済み）。

<!-- {{PROJECT_PITFALLS}} — PJ固有の落とし穴があれば以下に追記 -->
⚠️ sync-rules placeholder — PJ固有の落とし穴があれば追記すること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{PROJECT_PITFALLS}} -->
