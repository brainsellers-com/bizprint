---
version: "2.0.0"
has_placeholders: true
description: "テスト要件（汎用枠組み + PJ固有テストスタック）"
paths:
  - "**/src/test/**"
  - "**/tests/**"
  - "**/__tests__/**"
  - "**/*.test.*"
  - "**/*.spec.*"
---

# テスト要件

## 汎用ルール

- テストクラス/ファイルの命名は対象クラス/モジュール名 + Test（または .test / .spec）で統一する
- 既存テストのパターンを踏襲すること: 実装開始前に同モジュール内の既存テストを確認し、構造・Mock・アサーション方法を合わせる
- Arrange-Act-Assert パターンで構成すること

## PJ 固有テスト要件

<!-- {{TEST_REQUIREMENTS}} — PJ固有のテストスタック・カバレッジ基準・フレームワーク設定を記載 -->
<!-- 例（Java/Spring PJ の場合）:
### テストフレームワーク
- JUnit 5 (`org.junit.jupiter.api.*`)
- Mockito: `@Mock`, `@InjectMocks`, `@MockBean`
- Spring Boot Test: `@SpringBootTest`, `@WebMvcTest`, MockMvc
- AssertJ（推奨）: `assertThat()`

### アノテーション
- @Test: テストメソッドに必須
- @DisplayName: 日本語の説明を追加
- @ParameterizedTest: 複数パターンのテストに使用

### カバレッジ基準
- 行カバレッジ: 80%以上（JaCoCo）
- 分岐カバレッジ: 70%以上（JaCoCo）
- 除外対象: 自動生成クラス、DTOクラス（getter/setterのみ）、定数クラス
-->
⚠️ sync-rules placeholder — PJ固有のテスト要件に置き換えること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{TEST_REQUIREMENTS}} -->
