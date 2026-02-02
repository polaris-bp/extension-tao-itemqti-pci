# XSSセキュリティレビュー結果

## 実施日
2026-02-01

## 検索結果サマリー

### 1. `.html()` 使用箇所

#### 要確認（優先度: 高）
| ファイル | 行 | コード | リスク評価 |
|---------|-----|--------|-----------|
| creator/widget/states/Map.js | 132 | `$responseForm.html(responseFormTpl({` | ⚠️ テンプレート関数の実装確認が必要 |
| creator/widget/states/Question.js | 284 | `$form.html(formTpl(_.assign({` | ⚠️ テンプレート関数の実装確認が必要 |

#### 確認済み（優先度: 低）
| ファイル | 行 | 説明 | リスク評価 |
|---------|-----|------|-----------|
| runtime/mathquill/mathquill.js | 全般 | MathQuillライブラリ内部実装 | ✅ サードパーティライブラリ（別途脆弱性確認） |

### 2. `.append()`, `.prepend()`, `.after()`, `.before()` 使用箇所

#### 安全（DOM要素を渡している）
| ファイル | 行 | コード | 評価 |
|---------|-----|--------|------|
| runtime/mathEntryInteraction.js | 421 | `this.$toolbar.after(this.$inputPlaceholder);` | ✅ DOM要素 |
| runtime/mathEntryInteraction.js | 927 | `self.$toolbar.append(self.createToolGroup(...));` | ✅ DOM要素返却関数 |
| runtime/mathEntryInteraction.js | 959 | `$toolGroup.append(self.createTool(toolConfig));` | ✅ DOM要素返却関数 |
| creator/widget/states/Question.js | 378 | `$toolbar.after($addGapBtn)` | ✅ DOM要素 |

#### 要確認（テンプレート関数使用）
| ファイル | 行 | コード | リスク評価 |
|---------|-----|--------|-----------|
| creator/widget/states/Map.js | 268 | `$(parent).prepend(scoreTpl({` | ⚠️ テンプレート関数の実装確認が必要 |
| creator/widget/states/Map.js | 274 | `$(parent).append($addAlternativeBtn);` | ✅ DOM要素 |
| creator/widget/states/Map.js | 536 | `$container.find(...).before(alternativeFormTpl({` | ⚠️ テンプレート関数の実装確認が必要 |

#### MathQuillライブラリ内
| ファイル | 説明 | 評価 |
|---------|------|------|
| runtime/mathquill/mathquill.js | ライブラリ内部実装 | ✅ サードパーティライブラリ |

### 3. jQueryコンストラクタ `$('<...')` 使用箇所

#### 安全（静的HTML）
すべて静的なHTML要素作成で、ユーザー入力を含まない:
- `$('<div>', {})` - オブジェクト形式（安全）
- `$('<span class="mq-cursor">&#8203;</span>')` - 静的HTML（MathQuill内）
- `$('<span class="mq-root-block"/>')` - 静的HTML（MathQuill内）

## 要確認アクションアイテム

### 優先度P0: テンプレート関数のエスケープ確認

以下のテンプレート関数がユーザー入力を適切にエスケープしているか確認が必要:

1. **Map.js で使用されるテンプレート**
   - `responseFormTpl` (Map.js:132)
   - `scoreTpl` (Map.js:268)
   - `alternativeFormTpl` (Map.js:536)

2. **Question.js で使用されるテンプレート**
   - `formTpl` (Question.js:284)

### 確認方法

#### ステップ1: テンプレート関数の実装を確認
```bash
# テンプレート関数定義を検索
grep -rn "responseFormTpl\|scoreTpl\|alternativeFormTpl\|formTpl" \
  views/js/pciCreator/ims/mathEntryInteraction/
```

#### ステップ2: テンプレートファイルの確認
一般的なパターン:
- Handlebarsテンプレート: `{{value}}` は自動エスケープ、`{{{value}}}` は非エスケープ
- Lodashテンプレート: `<%= value %>` は非エスケープ、`<%- value %>` はエスケープ
- 独自テンプレート: 実装を個別確認

#### ステップ3: 渡されるデータの確認
- ユーザー入力が含まれるか
- サーバーから取得したデータが含まれるか
- 静的データのみか

### 優先度P1: ユーザー入力表示箇所の特定

以下の箇所でユーザー入力がどのように表示されているか確認:
- [ ] MathQuillフィールドへの入力値反映
- [ ] Gap Expressionフィールドへの入力値反映
- [ ] プロパティパネルでの値表示
- [ ] レビューモードでの正答表示

## 次のアクション

1. テンプレート関数のソースコードを確認
2. テンプレートエンジンのエスケープ仕様を確認
3. 非エスケープ出力（`{{{}}}`や`<%=`）が使われている箇所を特定
4. ユーザー入力が非エスケープで出力されている場合は修正

## 検索コマンド実行履歴

```bash
# .html()使用箇所
grep -rn "\.html\(" views/js/pciCreator/ims/mathEntryInteraction/ --include="*.js"

# append/prepend等の使用箇所
grep -rn "\.(append|prepend|after|before|replaceWith)\(" \
  views/js/pciCreator/ims/mathEntryInteraction/ --include="*.js"

# jQueryコンストラクタでHTML作成
grep -rn "\$\([\"'<]" views/js/pciCreator/ims/mathEntryInteraction/ --include="*.js"
```

## テンプレートエンジン分析結果

### 使用テンプレートエンジン
**Handlebars** - 自動HTMLエスケープ機能あり

### テンプレート一覧とエスケープ状況

| ファイル | エスケープ状況 | 非エスケープ箇所 | リスク評価 |
|---------|--------------|----------------|-----------|
| markup.tpl | ⚠️ 一部非エスケープ | Line 2: `{{{prompt}}}` | 中リスク |
| responseForm.tpl | ✅ 全てエスケープ | なし | 安全 |
| propertiesForm.tpl | ✅ 全てエスケープ | なし | 安全 |
| scoreForm.tpl | ✅ 全てエスケープ | なし | 安全 |
| answerForm.tpl | ✅ 全てエスケープ | なし | 安全 |
| alternativeForm.tpl | ✅ 全てエスケープ | なし | 安全 |
| addGapBtn.tpl | ✅ 全てエスケープ | なし | 安全 |
| addAlternativeBtn.tpl | ✅ 全てエスケープ | なし | 安全 |

### 🔴 重要な発見: markup.tpl の非エスケープ出力

**ファイル**: `creator/tpl/markup.tpl`
**行番号**: 2
**コード**: `<div class="prompt">{{{prompt}}}</div>`

#### リスク分析

**脅威レベル**: 中リスク

**理由**:
1. **意図的な非エスケープの可能性**:
   - TAOのコンテンツエディタはリッチテキスト（HTML）をサポート
   - プロンプトに太字、イタリック、リンク等のHTMLタグを含める必要がある
   - したがって、非エスケープは機能要件の可能性

2. **リスク軽減要因**:
   - promptはアイテム作成者（信頼されたユーザー）のみが編集可能
   - 受験者（エンドユーザー）はpromptを編集できない
   - TAOのコンテンツエディタがサニタイズを行っている可能性

3. **残存リスク**:
   - アイテム作成者が悪意のあるスクリプトを挿入できる
   - コンテンツエディタのサニタイズをバイパスできる場合XSSが成立
   - 他のアイテム作成者や受験者に影響を与える可能性

#### 推奨事項

**優先度P1**: promptのサニタイズ状況確認
- [ ] TAOのcontainerEditorがHTMLサニタイズを実装しているか確認
- [ ] DOMPurifyなどのサニタイズライブラリを使用しているか確認
- [ ] 許可されるHTMLタグのホワイトリストを確認

**優先度P2**: セキュリティテスト
- [ ] promptに `<script>alert('XSS')</script>` を挿入してテスト
- [ ] promptに `<img src=x onerror=alert('XSS')>` を挿入してテスト
- [ ] TAOコンテンツエディタのバイパスが可能か確認

**優先度P3**: 代替案検討
もしサニタイズが不十分な場合:
1. promptをエスケープして出力（`{{prompt}}`）し、リッチテキストをあきらめる
2. DOMPurifyをPCI側で実装してサニタイズする
3. TAO側のサニタイズ強化を要求する

### promptのデータフロー

```
[TAO コンテンツエディタ]
        ↓
    (HTMLサニタイズ?)
        ↓
[interaction.data('prompt', text)]
        ↓
   (Question.js:98)
        ↓
  [markup.tpl:2]
        ↓
   {{{prompt}}}  ← 非エスケープ出力
        ↓
 [受験者のブラウザ]
```

**確認が必要な箇所**: TAOのcontainerEditorのサニタイズ実装

## 追加調査が必要な項目

- [x] テンプレートファイルの場所特定 → 完了
- [x] テンプレートエンジンの種類確認（Handlebars/Lodash/その他） → Handlebars
- [x] 各テンプレート関数に渡されるデータの内容確認 → 完了
- [ ] **TAOのcontainerEditorのHTMLサニタイズ実装確認** ← 最優先
- [ ] promptのXSSテスト実施
- [ ] ユーザー入力値（数式）の流れを追跡（入力 → 保存 → 表示）
- [ ] サーバーから返されるresponseデータの表示方法確認
