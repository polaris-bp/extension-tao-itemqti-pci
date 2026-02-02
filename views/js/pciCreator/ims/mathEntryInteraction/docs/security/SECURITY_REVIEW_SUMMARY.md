# Math Entry Interaction - セキュリティレビューサマリー

## 実施日
2026-02-01

## レビュー範囲
- Math Entry Interaction PCI全体
- 特にjQueryメソッドとHandlebarsテンプレートのXSS脆弱性

## エグゼクティブサマリー

### 総合評価
**中リスク（1件の要調査項目あり）**

### 主な発見事項
1. ✅ jQuery危険メソッド（`.html()`等）の使用は概ね安全
2. ⚠️ Handlebarsテンプレート1箇所で非エスケープ出力を発見
3. ✅ ユーザー入力の直接的なHTML挿入はなし

---

## 詳細レビュー結果

### 1. jQuery危険メソッド分析

#### 質問: 危険なjQueryメソッドは`.html()`だけですか？

**回答**: いいえ、以下のメソッドもXSSリスクがあります。

#### 危険なjQueryメソッド一覧

| カテゴリ | メソッド | リスクレベル | 条件 |
|---------|---------|------------|------|
| 高リスク | `.html(str)` | 🔴 高 | ユーザー入力含む場合常に危険 |
| 高リスク | `$(htmlString)` | 🔴 高 | ユーザー入力含む場合常に危険 |
| 中リスク | `.append(str)` | 🟡 中 | HTML文字列+ユーザー入力の場合のみ |
| 中リスク | `.prepend(str)` | 🟡 中 | HTML文字列+ユーザー入力の場合のみ |
| 中リスク | `.after(str)` | 🟡 中 | HTML文字列+ユーザー入力の場合のみ |
| 中リスク | `.before(str)` | 🟡 中 | HTML文字列+ユーザー入力の場合のみ |
| 中リスク | `.replaceWith(str)` | 🟡 中 | HTML文字列+ユーザー入力の場合のみ |
| 低リスク | `.attr('onclick', ...)` | 🟡 中 | インラインイベントハンドラ |
| 低リスク | `.attr('href', 'javascript:...')` | 🟡 中 | javascript:スキーム |

#### mathEntryInteractionでの使用状況

| メソッド | 使用箇所数 | 評価 | 備考 |
|---------|----------|------|------|
| `.html()` | 2箇所（PCI内） | ✅ 安全 | Handlebarsテンプレート使用 |
| `.append()` | 3箇所 | ✅ 安全 | DOM要素のみ渡している |
| `.prepend()` | 2箇所 | ✅ 安全 | DOM要素のみ渡している |
| `.after()` | 2箇所 | ✅ 安全 | DOM要素のみ渡している |
| `.before()` | 2箇所 | ✅ 安全 | DOM要素のみ渡している |
| その他危険メソッド | 0箇所 | ✅ 安全 | 使用なし |

**結論**: jQueryメソッドの使用は適切。HTML文字列+ユーザー入力の直接連結なし。

---

### 2. Handlebarsテンプレート分析

#### テンプレートエンジン
**Handlebars** - デフォルトで自動HTMLエスケープ

#### エスケープ仕様
- `{{value}}` → 自動エスケープ（安全）
- `{{{value}}}` → 非エスケープ（危険）

#### 分析結果

| テンプレートファイル | 非エスケープ箇所 | 評価 |
|------------------|----------------|------|
| markup.tpl | Line 2: `{{{prompt}}}` | ⚠️ 要調査 |
| responseForm.tpl | なし | ✅ 安全 |
| propertiesForm.tpl | なし | ✅ 安全 |
| scoreForm.tpl | なし | ✅ 安全 |
| answerForm.tpl | なし | ✅ 安全 |
| alternativeForm.tpl | なし | ✅ 安全 |
| addGapBtn.tpl | なし | ✅ 安全 |
| addAlternativeBtn.tpl | なし | ✅ 安全 |

---

### 3. 🔴 重要な発見: markup.tpl の非エスケープ出力

#### 問題箇所
**ファイル**: `creator/tpl/markup.tpl`
**行番号**: 2
**コード**:
```handlebars
<div class="prompt">{{{prompt}}}</div>
```

#### リスク評価

| 項目 | 評価 |
|-----|------|
| 脅威レベル | 🟡 中リスク |
| 影響範囲 | アイテム作成者・受験者 |
| 悪用難易度 | 中（アイテム作成者権限が必要） |
| CVSSスコア（推定） | 5.4 (Medium) |

#### リスク分析

**なぜ非エスケープを使用しているか**:
- TAOのコンテンツエディタはリッチテキスト（HTML）をサポート
- プロンプトに太字、イタリック、リンク等を含める必要がある
- したがって、`{{{prompt}}}`の使用は**機能要件**の可能性が高い

**リスク軽減要因**:
- ✅ promptはアイテム作成者（信頼されたユーザー）のみが編集可能
- ✅ 受験者（エンドユーザー）はpromptを直接編集不可
- ⚠️ TAOのcontainerEditorがサニタイズを行っているかは未確認

**残存リスク**:
- ❌ アイテム作成者が悪意のあるスクリプトを挿入可能
- ❌ コンテンツエディタのサニタイズ不足やバイパスが可能な場合XSSが成立
- ❌ 他のアイテム作成者や受験者に影響を与える可能性（Stored XSS）

#### データフロー

```
[TAOコンテンツエディタ]
        ↓
   (サニタイズ?)  ← 確認が必要
        ↓
[interaction.data('prompt', text)]
        ↓
  [Question.js:98]
        ↓
  [markup.tpl:2]
        ↓
  {{{prompt}}}  ← 非エスケープ
        ↓
[受験者のブラウザ]
```

---

## 推奨事項

### 優先度P0: 即座に実施すべき対策

なし（明確なXSS脆弱性は未発見）

### 優先度P1: 早急に確認すべき事項

#### 1. TAOのcontainerEditorのサニタイズ確認
- [ ] containerEditorのソースコード確認
- [ ] DOMPurifyなどのサニタイズライブラリ使用の有無
- [ ] 許可HTMLタグのホワイトリスト確認

#### 2. promptのXSSテスト実施
- [ ] `<script>alert('XSS')</script>` を挿入してテスト
- [ ] `<img src=x onerror=alert('XSS')>` を挿入してテスト
- [ ] `<svg onload=alert('XSS')>` を挿入してテスト
- [ ] エディタのバイパス可能性確認

### 優先度P2: 計画的に実施すべき対策

#### もしサニタイズが不十分な場合の対応策

**選択肢1**: promptをエスケープ（リッチテキストを諦める）
```handlebars
<!-- 変更前 -->
<div class="prompt">{{{prompt}}}</div>

<!-- 変更後 -->
<div class="prompt">{{prompt}}</div>
```
- ✅ 完全にXSSを防げる
- ❌ リッチテキスト（太字、イタリック等）が表示できなくなる

**選択肢2**: PCI側でDOMPurifyを実装
```javascript
// DOMPurifyをインストール
// npm install dompurify --save

// テンプレートに渡す前にサニタイズ
import DOMPurify from 'dompurify';
const cleanPrompt = DOMPurify.sanitize(prompt);
```
- ✅ リッチテキストを維持できる
- ✅ XSSを防げる
- ❌ 依存関係が増える

**選択肢3**: TAO側のサニタイズ強化を要求
- ✅ 根本的な解決
- ✅ 他のPCIにも恩恵
- ❌ TAO開発チームの対応が必要

### 優先度P3: 継続的な対策

1. **定期的なセキュリティレビュー**
   - 新規テンプレート追加時に`{{{}}}`使用チェック
   - jQuery危険メソッド使用時のレビュー

2. **自動化**
   - pre-commit hookで`{{{`を検出
   - ESLintでjQuery危険メソッドを警告

---

## チェックリスト

実装済みのチェックリスト:
- [x] jQuery危険メソッドの一覧化
- [x] 全テンプレートファイルの非エスケープ出力検索
- [x] `.html()`使用箇所の静的解析
- [x] `.append()/.prepend()`等の使用箇所分析
- [x] promptデータフローの特定

未実施のチェックリスト:
- [ ] TAO containerEditorのサニタイズ実装確認
- [ ] promptのXSSペネトレーションテスト
- [ ] MathQuillフィールド入力値のXSSテスト
- [ ] Gap Expressionフィールド入力値のXSSテスト
- [ ] 数式レスポンスデータ表示のXSSテスト

---

## 関連ドキュメント

- [jQuery XSSセキュリティチェックリスト](./jquery-xss-checklist.md)
- [XSSセキュリティレビュー詳細](./xss-review-findings.md)

---

## 結論

### 現状評価
Math Entry Interaction PCIは、jQueryの危険メソッドを適切に使用しており、直接的なXSS脆弱性は発見されませんでした。

### 残存リスク
`markup.tpl`の`{{{prompt}}}`が唯一のリスク箇所です。このリスクはTAOのcontainerEditorのサニタイズ実装に依存しており、**早急な確認が必要**です。

### 次のアクション
1. TAO containerEditorのサニタイズ実装を確認
2. promptのXSSテストを実施
3. 必要に応じてDOMPurifyの導入を検討

---

## 承認

| 役割 | 氏名 | 日付 | 署名 |
|-----|------|------|------|
| セキュリティレビュー実施者 | | 2026-02-01 | |
| 承認者 | | | |
