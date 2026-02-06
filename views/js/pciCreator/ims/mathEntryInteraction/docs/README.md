# Math Entry Interaction - ドキュメント

## 概要

Math Entry Interaction PCI の包括的なドキュメントセット。設計書、セキュリティレビュー、ソースコード解説を含みます。

## ドキュメント構成

### 📘 設計ドキュメント（日本語）

**ディレクトリ**: `design/ja/`

| ドキュメント | 説明 | 対象読者 |
|-----------|------|---------|
| [01_概要設計書.md](./design/ja/01_概要設計書.md) | システム概要、機能要件、非機能要件 | プロジェクトマネージャー、アーキテクト |
| [02_アーキテクチャ設計書.md](./design/ja/02_アーキテクチャ設計書.md) | アーキテクチャ、コンポーネント構成、データフロー | アーキテクト、シニア開発者 |
| [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) | Runtimeモジュールの詳細仕様 | 開発者 |
| [04_Creatorモジュール仕様書.md](./design/ja/04_Creatorモジュール仕様書.md) | Creatorモジュールの詳細仕様 | 開発者 |
| [05_インターフェース仕様書.md](./design/ja/05_インターフェース仕様書.md) | IMS PCI準拠インターフェース仕様 | 開発者、QA |

### 🔒 セキュリティドキュメント

**ディレクトリ**: `security/`

**概要**: XSS脆弱性、jQuery/Lodash CVE、ライブラリバージョン調査

| ドキュメント | 説明 | 対象読者 |
|-----------|------|---------|
| [README.md](./security/README.md) | セキュリティドキュメント総合インデックス | 全員 |
| [SECURITY_REVIEW_SUMMARY.md](./security/SECURITY_REVIEW_SUMMARY.md) | XSSセキュリティレビュー総合サマリー | PM、セキュリティ担当者 |
| [jquery-xss-checklist.md](./security/jquery-xss-checklist.md) | jQuery XSS対策実務チェックリスト | 開発者、レビュアー |
| [xss-review-findings.md](./security/xss-review-findings.md) | XSS脆弱性詳細調査結果 | セキュリティエンジニア |
| [CVE_BASED_CHECKLIST.md](./security/CVE_BASED_CHECKLIST.md) | jQuery/Lodash CVEベース確認チェックリスト | 開発者、監査担当者 |
| [TAO_JQUERY_VERSION_REPORT.md](./security/TAO_JQUERY_VERSION_REPORT.md) | TAO Platform jQuery調査レポート | アーキテクト、PM |
| [LODASH_SECURITY_REPORT.md](./security/LODASH_SECURITY_REPORT.md) | Lodash脆弱性調査レポート | アーキテクト、PM |

### 💻 ソースコード解説

**ファイル**: `SOURCE_CODE_GUIDE.md`

| ドキュメント | 説明 | 対象読者 |
|-----------|------|---------|
| [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) | ファイル構成と各ファイルの役割解説 | 開発者、新規参加者 |

---

## クイックスタート

### 🎯 目的別ガイド

#### 「全体像を理解したい」
1. **設計書**: [01_概要設計書.md](./design/ja/01_概要設計書.md) を読む
2. **アーキテクチャ**: [02_アーキテクチャ設計書.md](./design/ja/02_アーキテクチャ設計書.md) を読む
3. **ソースコード**: [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) でファイル構成を把握

#### 「開発・カスタマイズしたい」
1. **ソースコード**: [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) で重要ファイルを特定
2. **Runtime仕様**: [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) で実装詳細を確認
3. **Creator仕様**: [04_Creatorモジュール仕様書.md](./design/ja/04_Creatorモジュール仕様書.md) で編集機能を理解

#### 「セキュリティ確認したい」
1. **総合サマリー**: [security/README.md](./security/README.md) でリスク評価を確認
2. **XSS対策**: [security/jquery-xss-checklist.md](./security/jquery-xss-checklist.md) でチェックリスト実施
3. **CVE確認**: [security/CVE_BASED_CHECKLIST.md](./security/CVE_BASED_CHECKLIST.md) で既知脆弱性を確認

#### 「IMS PCI仕様を知りたい」
1. **インターフェース**: [05_インターフェース仕様書.md](./design/ja/05_インターフェース仕様書.md) を読む
2. **公式仕様**: IMS PCI v1.0 仕様書を参照

---

## ドキュメント更新履歴

| 日付 | バージョン | 更新内容 | 担当者 |
|-----|----------|---------|-------|
| 2026-01-11 | 1.0.0 | 設計ドキュメント作成（5ファイル） | Claude Code |
| 2026-02-01 | 1.1.0 | XSSセキュリティレビュー追加 | Claude Code |
| 2026-02-02 | 1.2.0 | jQuery調査レポート追加 | Claude Code |
| 2026-02-02 | 1.3.0 | Lodash調査レポート追加 | Claude Code |
| 2026-02-02 | 1.4.0 | CVEベースチェックリスト追加 | Claude Code |
| 2026-02-02 | 1.5.0 | ソースコードガイド追加 | Claude Code |

---

## 主要トピック別索引

### 機能仕様

- **数式入力**: [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) - `initField()`
- **ツールバー**: [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) - `initToolbar()`
- **Gap Expression**: [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) - Gap関連メソッド
- **正答設定**: [04_Creatorモジュール仕様書.md](./design/ja/04_Creatorモジュール仕様書.md) - Correct状態
- **採点設定**: [04_Creatorモジュール仕様書.md](./design/ja/04_Creatorモジュール仕様書.md) - Map状態

### セキュリティ

- **XSS対策チェックリスト**: [security/jquery-xss-checklist.md](./security/jquery-xss-checklist.md)
- **jQuery CVE確認**: [security/CVE_BASED_CHECKLIST.md](./security/CVE_BASED_CHECKLIST.md) - jQuery 2.1.1セクション
- **Lodash CVE確認**: [security/CVE_BASED_CHECKLIST.md](./security/CVE_BASED_CHECKLIST.md) - Lodash 4.17.21セクション
- **Handlebars非エスケープ**: [security/xss-review-findings.md](./security/xss-review-findings.md) - markup.tpl分析

### アーキテクチャ

- **コンポーネント構成**: [02_アーキテクチャ設計書.md](./design/ja/02_アーキテクチャ設計書.md) - 第3章
- **データフロー**: [02_アーキテクチャ設計書.md](./design/ja/02_アーキテクチャ設計書.md) - 第4章
- **デザインパターン**: [02_アーキテクチャ設計書.md](./design/ja/02_アーキテクチャ設計書.md) - 第6章
- **依存関係**: [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) - 依存関係図

### 実装詳細

- **ファイル構成**: [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) - ディレクトリ構造
- **主要クラス**: [03_Runtimeモジュール仕様書.md](./design/ja/03_Runtimeモジュール仕様書.md) - クラス一覧
- **状態管理**: [04_Creatorモジュール仕様書.md](./design/ja/04_Creatorモジュール仕様書.md) - State Pattern
- **ビルドプロセス**: [SOURCE_CODE_GUIDE.md](./SOURCE_CODE_GUIDE.md) - ビルド関連

---

## 統計情報

### ドキュメント規模

| カテゴリ | ファイル数 | 総ページ数（推定） |
|---------|----------|----------------|
| 設計ドキュメント | 5 | 約100ページ |
| セキュリティ | 7 | 約150ページ |
| ソースコード解説 | 1 | 約25ページ |
| **合計** | **13** | **約275ページ** |

### ソースコード規模（参考）

| カテゴリ | ファイル数 | 総行数 |
|---------|----------|-------|
| Runtime JavaScript | 6 | 約7,000行 |
| Creator JavaScript | 5 | 約1,300行 |
| Handlebarsテンプレート | 8 | 約400行 |
| CSS | 2 | 約600行 |
| 設定ファイル | 3 | 約150行 |
| **合計** | **24** | **約9,450行** |

---

## 関連リソース

### 外部仕様

- [IMS PCI v1.0 仕様](https://www.imsglobal.org/question/qtiv2p1/imsqti_intguide_v2p1.html)
- [QTI 2.1 仕様](https://www.imsglobal.org/question/qtiv2p1/imsqti_infov2p1.html)

### 使用ライブラリ

- [MathQuill](http://mathquill.com/) - v0.10.2（2014年）
- [jQuery](https://jquery.com/) - 2.1.1（TAO提供）
- [Lodash](https://lodash.com/) - 4.17.21（TAO提供）
- [mathml-to-latex](https://www.npmjs.com/package/mathml-to-latex) - 1.3.0

### TAO関連

- [TAO Platform](https://www.taotesting.com/)
- [oat-sa/tao-core](https://github.com/oat-sa/tao-core)
- [oat-sa/extension-tao-itemqti](https://github.com/oat-sa/extension-tao-itemqti)
- [oat-sa/extension-tao-itemqti-pci](https://github.com/oat-sa/extension-tao-itemqti-pci)

---

## ドキュメント作成方針

### 日本語設計書の特徴

- **網羅性**: 全機能を漏れなく記載
- **詳細性**: メソッドレベルの詳細仕様を含む
- **実用性**: コード例、シーケンス図、状態遷移図を多用
- **保守性**: バージョン管理、変更履歴の記録

### セキュリティドキュメントの特徴

- **実務的**: 検索コマンド、チェックリスト形式
- **CVEベース**: 既知脆弱性に基づく確認項目
- **リスク評価**: mathEntryInteraction固有のリスク分析
- **アクションプラン**: 優先度付き推奨事項

### ソースコードガイドの特徴

- **階層的**: ディレクトリ構造から詳細へ
- **役割明確**: 各ファイルの責務を明記
- **重要度**: ファイル重要度ランキング
- **依存関係**: 明示的な依存関係図

---

## フィードバック・貢献

### ドキュメント改善提案

ドキュメントの誤り、不明瞭な点、追加すべき内容がありましたら、以下の方法でフィードバックをお願いします:

1. **GitHub Issue**: リポジトリにIssueを起票
2. **Pull Request**: 直接修正してPRを作成
3. **社内レビュー**: チーム内で議論して改善

### 更新ガイドライン

ドキュメントを更新する際は:

1. **変更履歴**: 必ず更新履歴テーブルに記録
2. **バージョン**: セマンティックバージョニング
3. **一貫性**: 既存のフォーマット・用語に従う
4. **検証**: 技術的正確性を確認

---

## ライセンス

このドキュメントは Math Entry Interaction PCI と同じライセンス（GPL-2.0）で提供されます。

---

**最終更新**: 2026-02-02
**ドキュメントバージョン**: 1.5.0
**PCIバージョン**: 2.8.0
