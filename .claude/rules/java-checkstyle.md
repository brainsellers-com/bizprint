---
version: "1.1.1"
has_placeholders: false
description: "Java CheckStyle 準拠ルール（コミット前確認・ボーイスカウトルール・違反パターン・EditorConfig）"
paths:
  - "**/*.java"
  - "bizprint-server-java/**/*.java"
  - "config/checkstyle/**"
---

# Java コーディングルール（CheckStyle 準拠）

> このルールは `paths: ["**/*.java"]` により Java ファイル変更時に自動適用される。EditorConfig セクションは非 Java ファイルにも関係するが、Java 変更のついでに確認する運用を想定している。

## MUST
- Java コードの追加・修正は `config/checkstyle/checkstyle.xml` に準拠すること。
- 規約に迷った場合は `config/checkstyle/checkstyle.xml` を正（source of truth）とする。
- 変更後は CheckStyle 違反 0 件で提出すること。

## コミット前の確認手順

Java ソースを変更した場合、コミット前に以下を実行する:

```powershell
mvn checkstyle:check -pl <module> -Dcheckstyle.consoleOutput=true
```

違反が報告された場合は修正してからコミットする。

## ボーイスカウトルール

ファイルを修正したら、**そのファイル全体**を CheckStyle 準拠にする:

1. 未使用 import の削除
2. import 順序の整理（java → javax → org → com.*）
3. スター import の排除（static import は可）
4. インデントの統一（スペース4）
5. 中括弧スタイルの統一（K&R: 開き括弧は行末）
6. snake_case フィールドの camelCase リネーム

## よくある違反パターンと対処法

| 違反 | 対処 |
|---|---|
| `MemberName` / `LocalVariableName` | snake_case → camelCase にリネーム |
| `ImportOrder` | IDE の Optimize Imports を使用するか手動で並べ替え |
| `UnusedImports` | 不要な import を削除 |
| `AvoidStarImport` | `import java.util.*` → 個別 import に展開 |
| `FileTabCharacter` | タブ → スペース4に変換 |
| `LeftCurly` | 中括弧を行末に移動（K&R スタイル） |
| `WhitespaceAround` | 演算子の前後にスペースを追加 |
| `ConstantName` | 定数は UPPER_CASE にリネーム |
| `MethodName` | public API メソッドはリネーム不可（[下記参照](#public-api-メソッド名の変更禁止)） |

## snake_case → camelCase リネーム時の注意

- **フィールド名のみリネームする。** 外部入力からの自動マッピング（XML パーサ、DI フレームワーク等）で setter 名が使われている場合、setter メソッド名は変更不可。
- 例: `private int page_count;` → `private int pageCount;`（setter `setPage_count()` はそのまま）
- リネーム後はコンパイルが通ることを確認する。

## public API メソッド名の変更禁止

ライブラリモジュールの **public メソッドはユーザーが直接呼び出す公開 API** であるため、メソッド名を変更してはいけない。CheckStyle の `MethodName` 違反が報告されても、public メソッドはリネームせずそのままにする。

- 対象: ライブラリとして公開しているモジュールの public メソッド
- private / package-private / protected メソッドはリネーム可

## 抑制ルール（suppressions.xml）

- `ConstantName` の `logger` パターン: package-private logger はプロジェクト慣習として許容

## EditorConfig（非 Java ファイル）

非 Java ファイルの変更時も EditorConfig に従う:

```powershell
mvn editorconfig:check -pl <module>
```

- 文字コード: UTF-8
- 改行: `.editorconfig` の `end_of_line` に従う
- 末尾空白の除去、最終行に改行

## Quick tips

- マジックナンバーは定数化（`static final`）。

## Reference
- 詳細: `config/checkstyle/checkstyle.xml`
