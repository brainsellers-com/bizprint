---
version: "1.1.0"
has_placeholders: true
description: "セキュリティ制約（機密情報管理・認証・バリデーション）"
paths:
  - "**/src/main/java/**/*.java"
---

# セキュリティ制約

本プロジェクトにおけるセキュリティ関連の必須ルール。

## MUST NOT

- APIキー、パスワード、個人情報などの機密情報を、いかなる形式でも出力してはいけない
- APIキー、パスワード、個人情報などの機密情報を、DBに格納するコードを作成してはいけない
- 機密情報を平文でログ出力してはいけない
- 機密情報をソースコードに直接埋め込んではいけない
- 機密情報をコミットしてはいけない（.env、credentials.json等）

## MUST

- APIキー管理は設定ファイルまたは環境変数で安全に管理すること
<!-- {{API_KEY_CONFIG}} — APIキー設定ファイルのパス。例: config/api-keys.properties -->
⚠️ sync-rules placeholder — PJ固有のAPIキー設定ファイルパスに置き換えること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{API_KEY_CONFIG}} -->
- ロール・パーミッション検証を常に確認すること（権限割当テーブルの設計・実装時の注意点を意識）
- 認証・認可を適切に実装すること
<!-- {{AUTH_MODEL}} — PJ固有の認証モデル。例: フロント側: Cognito + JWT + ロール・パーミッション、バック側: APIキー認証（X-API-Keyヘッダ） -->
⚠️ sync-rules placeholder — PJ固有の認証モデルに置き換えること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{AUTH_MODEL}} -->
- 入力値は必ずバリデーションすること
  - SQLインジェクション対策
  - XSS対策
  - パストラバーサル対策

## ログ出力時のマスキング

機密情報をログ出力する場合は、必ずマスキング処理を実施すること：

```java
// NG: 機密情報を平文で出力
logger.info("API Key: {}", apiKey);

// OK: マスキング処理を実施
logger.info("API Key: {}****", apiKey.substring(0, 8));
```

## 設定ファイルの管理

```properties
# APIキー設定ファイル
# このファイルは .gitignore に追加すること
api-key.valid-keys[0].key=${API_KEY_PRODUCTION}
api-key.valid-keys[0].name=Production Key
api-key.valid-keys[0].status=ACTIVE
```

## 参照

<!-- {{SECURITY_REFERENCES}} — PJ固有のセキュリティ関連ドキュメントへの参照を記載 -->
<!-- 例:
- 詳細なセキュリティ要件: `docs/requirements/common.md` の「セキュリティ共通方針」
- APIキー管理の詳細: `docs/requirements/decisions/backend/api_key_overview.md`
-->
⚠️ sync-rules placeholder — PJ固有のセキュリティ参照ドキュメントに置き換えること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{SECURITY_REFERENCES}} -->
