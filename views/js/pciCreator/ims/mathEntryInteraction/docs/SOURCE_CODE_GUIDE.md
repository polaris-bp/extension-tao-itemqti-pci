# Math Entry Interaction - ソースコードファイル構成ガイド

## 概要

このドキュメントは、Math Entry Interaction PCI のソースコードファイル構成と各ファイルの役割を詳しく解説します。

## 目次

1. [ディレクトリ構造](#ディレクトリ構造)
2. [設定ファイル](#設定ファイル)
3. [Runtimeファイル（実行時）](#runtimeファイル実行時)
4. [Creatorファイル（編集時）](#creatorファイル編集時)
5. [ビルド関連](#ビルド関連)
6. [依存関係図](#依存関係図)

---

## ディレクトリ構造

```
mathEntryInteraction/
├── imsPciCreator.js              # Creator エントリーポイント
├── imsPciCreator.json            # PCI設定ファイル（メタデータ）
├── package.json                  # npm依存関係
├── package-lock.json             # npm依存関係ロック
├── rollup.config.js              # Rollupビルド設定
│
├── runtime/                      # 実行時（受験者モード）
│   ├── mathEntryInteraction.js   # Runtime メインモジュール ★最重要
│   ├── css/
│   │   └── mathEntryInteraction.css  # スタイルシート
│   ├── helper/
│   │   ├── ambiguousSymbols.js   # 記号正規化ヘルパー
│   │   └── mathInPrompt.js       # プロンプト内数式レンダリング
│   ├── mathquill/                # MathQuill v0.10.2 ライブラリ
│   │   ├── mathquill.js
│   │   ├── mathquill.css
│   │   └── font/                 # Symbolaフォント（数式記号用）
│   ├── mathml-to-latex/          # MathML→LaTeX変換
│   │   └── mathml-to-latex.js
│   └── polyfill/
│       └── es6-collections.js    # ES6 Map/Set ポリフィル
│
└── creator/                      # 編集時（アイテム作成者モード）
    ├── widget/
    │   ├── Widget.js             # Creator ウィジェットベース
    │   └── states/               # 状態管理
    │       ├── states.js         # 状態バンドル
    │       ├── Question.js       # Question状態 ★重要
    │       ├── Correct.js        # Correct状態
    │       └── Map.js            # Map状態（採点設定）
    └── tpl/                      # Handlebarsテンプレート
        ├── markup.tpl            # メインHTML構造
        ├── propertiesForm.tpl    # プロパティフォーム
        ├── addGapBtn.tpl         # Gap追加ボタン
        ├── addAlternativeBtn.tpl # 別解追加ボタン
        ├── alternativeForm.tpl   # 別解フォーム
        ├── answerForm.tpl        # 正答フォーム
        ├── responseForm.tpl      # レスポンスフォーム
        └── scoreForm.tpl         # スコアフォーム
```

---

## 設定ファイル

### imsPciCreator.json

**役割**: PCI（Portable Custom Interaction）のメタデータと設定を定義

**主な内容**:
- **基本情報**: typeIdentifier, label, version, author
- **response設定**: baseType=string, cardinality=single
- **runtime設定**: 実行時に必要なファイルリスト
- **creator設定**: 編集時に必要なファイルリスト
- **mediaFiles**: Symbolaフォントファイル（10種類）

**重要なポイント**:
```json
{
  "runtime": {
    "hook": "./runtime/mathEntryInteraction.min.js",  // ビルド済みファイル
    "libraries": [],  // 空配列（Portable Librariesを使用しない）
    "src": [/* ソースファイルリスト */]
  },
  "creator": {
    "hook": "./imsPciCreator.min.js",  // ビルド済みファイル
    "src": [/* ソースファイルリスト */]
  }
}
```

**ビルドプロセス**: `src` のファイルが `hook` のminファイルにバンドルされる

---

### package.json

**役割**: npm依存関係とビルドスクリプトの定義

**主な内容**:
```json
{
  "name": "mathEntryInteraction",
  "version": "2.8.0",
  "scripts": {
    "build": "rollup -c",
    "watch": "rollup -c -w"
  },
  "devDependencies": {
    "rollup": "^2.79.1",
    "rollup-plugin-copy": "^3.4.0",
    "@rollup/plugin-commonjs": "^23.0.2",
    "@rollup/plugin-node-resolve": "^15.0.1",
    "mathml-to-latex": "1.3.0"
  }
}
```

**注**: 実行時依存関係（jQuery, Lodash）はTAO Coreから提供されるため、`dependencies` は空

---

### rollup.config.js

**役割**: Rollupバンドラーの設定

**機能**:
1. **2つのバンドルを生成**:
   - `runtime/mathEntryInteraction.min.js` - Runtime用
   - `imsPciCreator.min.js` - Creator用
2. **プラグイン**:
   - `@rollup/plugin-node-resolve`: node_modules解決
   - `@rollup/plugin-commonjs`: CommonJS→ESM変換
   - `rollup-plugin-copy`: アセットファイルコピー
3. **外部化**: TAO提供のライブラリ（jquery, lodash等）は外部化

---

## Runtimeファイル（実行時）

受験者が実際に問題を解く際に使用されるファイル群。

### runtime/mathEntryInteraction.js ★最重要

**役割**: Math Entry Interaction のメインロジック

**行数**: 1,259行（minify前）

**主要機能**:

#### 1. IMS PCI インターフェース実装
```javascript
getInstance: function(dom, config, state)
initialize: function(id, dom, config, assetManager)
render: function()
getResponse: function()
setState: function(state)
```

#### 2. MathQuillフィールド管理
- **通常モード**: 単一の数式入力フィールド
- **Gap Expressionモード**: 複数の空欄フィールド

```javascript
// Line 386-450: MathQuillフィールド初期化
initField: function() {
    this.mathField = MQ.MathField(this.$mathInput.get(0), {
        spaceBehavesLikeTab: this.config.authorizeWhiteSpace,
        handlers: {
            edit: function(mathField) {
                // 編集時の処理
            }
        }
    });
}
```

#### 3. ツールバー生成（80+ボタン）
```javascript
// Line 881-1041: ツールバー生成
initToolbar: function() {
    // toolGroupsの定義（分数、平方根、指数、三角関数等）
    // 各ツールボタンの動的生成
}
```

**ツールボタン例**:
- `tool_frac`: 分数 `\frac{□}{□}`
- `tool_sqrt`: 平方根 `\sqrt{□}`
- `tool_exp`: 指数 `x^{□}`
- `tool_sin`, `tool_cos`, `tool_tan`: 三角関数

#### 4. Gap Expression（空欄式）サポート
```javascript
// Line 466-546: Gap管理
addGap: function(index, latex) {
    // 空欄フィールドを動的追加
}
removeGap: function(index) {
    // 空欄フィールドを削除
}
```

**Gap Expressionの例**:
```
問題: x = [GAP1] + [GAP2]
学生が各GAPに数値を入力
```

#### 5. レスポンス処理
```javascript
// Line 647-690: レスポンス取得
getResponse: function(inputId) {
    if (this.inGapMode()) {
        // Gap Expression: カンマ区切り文字列
        return { base: { string: "2,3,5" } };
    } else {
        // 通常モード: LaTeX文字列
        return { base: { string: "x^{2}+3x+5" } };
    }
}
```

#### 6. 記号正規化
```javascript
// Line 180-186: ambiguousSymbols使用
convertAmbiguousSymbols = require('mathEntryInteraction/runtime/helper/ambiguousSymbols');
// 全角数字→ASCII、各種マイナス記号→標準マイナスに変換
```

**依存関係**:
- `qtiCustomInteractionContext`
- `taoQtiItem/portableLib/jquery_2_1_1` (jQuery 2.1.1)
- `taoQtiItem/portableLib/lodash` (Lodash 4.17.21)
- `mathEntryInteraction/runtime/mathquill/mathquill` (MathQuill)
- `mathEntryInteraction/runtime/helper/ambiguousSymbols`
- `mathEntryInteraction/runtime/helper/mathInPrompt`

---

### runtime/helper/ambiguousSymbols.js

**役割**: 曖昧な記号の正規化

**行数**: 56行

**機能**:
```javascript
// 全角数字 → ASCII数字
'０' → '0', '１' → '1', ..., '９' → '9'

// 各種マイナス記号 → 標準マイナス
'−' → '-', '‐' → '-', '―' → '-', '-' → '-'
```

**使用箇所**: `mathEntryInteraction.js` のレスポンス取得時

**重要性**: ユーザーが異なる入力方法で同じ記号を入力した場合の正規化

---

### runtime/helper/mathInPrompt.js

**役割**: プロンプト内のMathML数式を美しく表示

**行数**: 61行

**機能**:
1. **MathML検出**: `<math>...</math>` タグを検索
2. **LaTeX変換**: MathML → LaTeX（mathml-to-latexライブラリ使用）
3. **MathQuill描画**: LaTeX → 美しい数式表示

**使用例**:
```html
<!-- 元のプロンプト（MathML） -->
<div class="prompt">
  <p>次の式を解きなさい: <math><mi>x</mi><mo>+</mo><mn>3</mn></math></p>
</div>

<!-- mathInPrompt.postRender() 適用後 -->
<div class="prompt">
  <p>次の式を解きなさい: <span class="mq-math-mode">x + 3</span></p>
</div>
```

**依存関係**:
- `mathml-to-latex` (MathML→LaTeX変換)
- `mathquill` (美しい数式レンダリング)

**使用箇所**:
- Runtime: 問題表示時
- Creator: プロンプト編集時のプレビュー

---

### runtime/mathquill/mathquill.js

**役割**: 数式エディタライブラリ（サードパーティ）

**バージョン**: v0.10.2（2014年リリース）

**行数**: 約5,500行

**主な機能**:
1. **MathField**: 編集可能な数式入力フィールド
2. **StaticMath**: 静的な数式表示（編集不可）
3. **LaTeX変換**: 入力内容をLaTeX形式で取得
4. **キーボード/マウス操作**: 数式編集のUI

**API使用例**:
```javascript
const MQ = MathQuill.getInterface(2);

// 編集可能フィールド
const mathField = MQ.MathField(element, {
    handlers: {
        edit: function() {
            console.log(mathField.latex());  // LaTeX取得
        }
    }
});

// 静的表示
MQ.StaticMath(element);
```

**注**: このライブラリは8年以上前のもので、メンテナンスされていない可能性あり

---

### runtime/mathml-to-latex/mathml-to-latex.js

**役割**: MathML形式をLaTeX形式に変換

**バージョン**: 1.3.0

**使用箇所**: `mathInPrompt.js` でプロンプト内の数式を変換

**変換例**:
```xml
<!-- MathML -->
<math>
  <mi>x</mi>
  <mo>+</mo>
  <mn>3</mn>
</math>

<!-- LaTeX -->
x + 3
```

---

### runtime/polyfill/es6-collections.js

**役割**: ES6の`Map`と`Set`のポリフィル

**理由**: 古いブラウザ（IE11等）対応

**使用箇所**: MathQuillライブラリが内部的に使用

---

### runtime/css/mathEntryInteraction.css

**役割**: Math Entry Interaction のスタイル定義

**主な内容**:
1. **ツールバーレイアウト**: ボタン配置、グリッド
2. **MathQuillフィールド**: 入力エリアのスタイル
3. **Gap Expression**: 空欄フィールドのサイズ（small/medium/large）
4. **アイコン**: ツールボタンのSVGアイコン

**重要なクラス**:
```css
.mathEntryInteraction .toolbar { /* ツールバー */ }
.math-entry-input { /* 入力フィールド */ }
.math-gap-small { /* 小サイズ空欄 */ }
.math-gap-medium { /* 中サイズ空欄 */ }
.math-gap-large { /* 大サイズ空欄 */ }
```

---

## Creatorファイル（編集時）

アイテム作成者が問題を編集する際に使用されるファイル群。

### imsPciCreator.js

**役割**: Creator のエントリーポイント

**行数**: 約30行

**機能**: TAOのカスタムインタラクション登録

```javascript
define([
    'mathEntryInteraction/creator/widget/Widget',
    'tpl!mathEntryInteraction/creator/tpl/markup'
], function(Widget, markupTpl) {
    'use strict';

    var mathEntryInteractionCreator = {
        getTypeIdentifier: function() {
            return 'mathEntryInteraction';
        },
        getWidget: function() {
            return Widget;
        },
        getMarkupTemplate: function() {
            return markupTpl;
        },
        getDefaultProperties: function(pci) {
            return { /* デフォルトプロパティ */ };
        }
    };

    qtiCustomInteractionContext.register(mathEntryInteractionCreator);
});
```

---

### creator/widget/Widget.js

**役割**: Creatorウィジェットのベースクラス

**行数**: 41行

**機能**:
1. **状態登録**: Question, Correct, Map 状態を登録
2. **初期化**: ウィジェット初期化処理
3. **CSSクラス追加**: `.tao-qti-creator-context` を追加

```javascript
MathEntryInteractionWidget.initCreator = function() {
    this.registerStates(states);  // 3つの状態を登録
    Widget.initCreator.call(this);
    // ...
};
```

---

### creator/widget/states/states.js

**役割**: 3つの編集状態をバンドル

**行数**: 29行

**3つの状態**:
1. **Question**: 問題設定（プロパティ、ツールボタン選択）
2. **Correct**: 正答設定
3. **Map**: 採点設定（スコア、別解）

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/factory',
    'taoQtiItem/qtiCreator/widgets/interactions/customInteraction/states/states',
    'mathEntryInteraction/creator/widget/states/Question',
    'mathEntryInteraction/creator/widget/states/Correct',
    'mathEntryInteraction/creator/widget/states/Map'
], function(factory, states) {
    return factory.createBundle(states, arguments);
});
```

---

### creator/widget/states/Question.js ★重要

**役割**: Question状態の実装（問題プロパティ設定）

**行数**: 約500行

**主要機能**:

#### 1. プロパティフォーム生成
```javascript
// Line 283-294: フォーム生成
$form.html(formTpl(_.assign({
    serial: response.serial,
    identifier: interaction.attr('responseIdentifier'),
    authorizeWhiteSpace: toBoolean(...),
    useGapExpression: toBoolean(...),
    // ... 80+ のツールボタン設定
}, getToolsInitValues(interaction))));
```

#### 2. ツールボタン管理（80+項目）
```javascript
// Line 36-89: ツール定義
const tools = {
    tool_frac: { props: [{ name: 'tool_frac', defaultValue: true }]},
    tool_sqrt: { props: [{ name: 'tool_sqrt', defaultValue: true }]},
    // ... 80+ のツール定義
};
```

#### 3. Gap Expressionモード切り替え
```javascript
// Line 305-323: Gap切り替えハンドラ
useGapExpression: function(i, value) {
    if (toBoolean(value, false)) {
        self.createAddGapBtn();  // Gap追加ボタン作成
        $gapStyleBox.show();      // Gapサイズ選択表示
    } else {
        self.removeAddGapBtn();   // Gap追加ボタン削除
        $gapStyleBox.hide();      // Gapサイズ選択非表示
    }
}
```

#### 4. MathFieldリスナー
```javascript
// Line 401-449: MathField監視
addMathFieldListener: function() {
    this.on('mathFieldReady', function(mathField) {
        // MathField編集時の処理
        mathField.on('edit', function() {
            // プレビュー更新
        });
    });
}
```

**依存関係**:
- `taoQtiItem/qtiCreator/widgets/states/factory`
- `taoQtiItem/qtiCreator/widgets/interactions/states/Question`
- `tpl!mathEntryInteraction/creator/tpl/propertiesForm`
- `tpl!mathEntryInteraction/creator/tpl/addGapBtn`

---

### creator/widget/states/Correct.js

**役割**: Correct状態の実装（正答設定）

**行数**: 約100行

**主要機能**:

#### 1. 正答MathField初期化
```javascript
// 正答入力用のMathFieldを作成
// 問題設定と同じツールボタンを使用
```

#### 2. 正答値の保存
```javascript
// MathFieldの内容をinteractionのプロパティに保存
interaction.prop('correctResponse', mathField.latex());
```

#### 3. Gap Expression対応
```javascript
// Gap Expressionモードの場合、各空欄の正答を保存
```

---

### creator/widget/states/Map.js

**役割**: Map状態の実装（採点・スコア設定）

**行数**: 約700行

**主要機能**:

#### 1. レスポンス処理設定
```javascript
// Line 80-150: レスポンスフォーム生成
render: function() {
    $responseForm.html(responseFormTpl({
        serial: this.widget.serial,
        // ...
    }));
}
```

#### 2. スコア設定
```javascript
// Line 230-295: スコアフォーム
// デフォルトスコア（正答時）の設定
// スコア入力フィールドの生成
```

#### 3. 別解（Alternative）管理
```javascript
// Line 330-450: 別解追加・削除
addAlternative: function() {
    // 新しい別解フォームを追加
}
removeAlternative: function(index) {
    // 指定された別解を削除
}
```

**別解の例**:
```
正答: x^2 + 2x + 1
別解1: (x+1)^2
別解2: (x+1)(x+1)
```

#### 4. Response Processing
```javascript
// responseDeclarationの更新
// 正答、別解、スコアの情報を保存
```

**依存関係**:
- `tpl!mathEntryInteraction/creator/tpl/responseForm`
- `tpl!mathEntryInteraction/creator/tpl/scoreForm`
- `tpl!mathEntryInteraction/creator/tpl/answerForm`
- `tpl!mathEntryInteraction/creator/tpl/alternativeForm`
- `tpl!mathEntryInteraction/creator/tpl/addAlternativeBtn`

---

## Creatorテンプレートファイル

Handlebars形式のHTMLテンプレート。

### creator/tpl/markup.tpl

**役割**: Math Entry Interaction のメインHTML構造

**内容**:
```handlebars
<div class="mathEntryInteraction">
    <div class="prompt">{{{prompt}}}</div>
    <div class="math-entry">
        <div class="toolbar"></div>
        <div>
            <span class="math-entry-input" data-allow-copy="true"></span>
        </div>
    </div>
</div>
```

**重要**: `{{{prompt}}}` は非エスケープ（リッチテキスト対応）

---

### creator/tpl/propertiesForm.tpl

**役割**: プロパティ設定フォーム

**主な要素**:
1. **Response Identifier**: レスポンスID設定
2. **Options**:
   - `authorizeWhiteSpace`: 空白許可
   - `useGapExpression`: Gap Expressionモード
   - `gapStyle`: Gap サイズ（small/medium/large）
3. **Tools**: 80+ のツールボタンON/OFF設定
   - `toggle_all`: 全ツール一括ON/OFF
   - グループ別（Basic, Functions, Greek, Geometry, etc.）

**使用箇所**: `Question.js` の state

---

### creator/tpl/addGapBtn.tpl

**役割**: Gap追加ボタン

**内容**:
```handlebars
<button class="btn-info math-entry-add-gap">
    <i class="adder"> </i> {{__ "Add a gap"}}
</button>
```

**使用箇所**: Gap Expressionモード有効時に表示

---

### creator/tpl/addAlternativeBtn.tpl

**役割**: 別解追加ボタン

**内容**:
```handlebars
<button class="math-entry-response-correct btn-info">
    <i class="adder"> </i> {{__ "Add alternative"}}
</button>
```

**使用箇所**: `Map.js` state

---

### creator/tpl/alternativeForm.tpl

**役割**: 別解入力フォーム

**主な要素**:
- 別解入力用のMathFieldプレースホルダー
- 削除ボタン
- 別解のインデックス表示

**使用箇所**: `Map.js` で動的に生成・追加

---

### creator/tpl/answerForm.tpl

**役割**: 正答入力フォーム

**主な要素**:
- 正答入力用のMathFieldプレースホルダー
- "Correct Answer" ラベル

**使用箇所**: `Correct.js` と `Map.js`

---

### creator/tpl/responseForm.tpl

**役割**: レスポンス処理設定フォーム

**主な要素**:
- Response template 選択
- Response processing 設定

**使用箇所**: `Map.js` state

---

### creator/tpl/scoreForm.tpl

**役割**: スコア設定フォーム

**主な要素**:
- "Correct" ラベル
- スコア入力フィールド（数値、空欄可）

**使用箇所**: `Map.js` で正答・別解それぞれに表示

---

## ビルド関連

### ビルドプロセス

```bash
# ビルド実行
npm run build

# 監視モード（ファイル変更時に自動ビルド）
npm run watch
```

### 生成されるファイル

1. **`runtime/mathEntryInteraction.min.js`** (約500KB)
   - Runtime関連の全ファイルをバンドル
   - MathQuill, mathml-to-latex, helpers を含む
   - jQuery, Lodashは外部化（TAO Coreから読み込み）

2. **`imsPciCreator.min.js`** (約600KB)
   - Creator関連の全ファイルをバンドル
   - Widget, States, Templates を含む
   - Runtime機能も含む（プレビュー用）

### Rollup設定のポイント

```javascript
// rollup.config.js
export default [
  {
    input: 'runtime/mathEntryInteraction.js',
    output: {
      file: 'runtime/mathEntryInteraction.min.js',
      format: 'amd',  // TAOのAMDシステムに適合
    },
    external: [
      'qtiCustomInteractionContext',
      'taoQtiItem/portableLib/jquery_2_1_1',  // jQuery外部化
      'taoQtiItem/portableLib/lodash',        // Lodash外部化
      // ...
    ]
  },
  {
    input: 'imsPciCreator.js',
    output: {
      file: 'imsPciCreator.min.js',
      format: 'amd',
    },
    external: [/* 同上 */]
  }
];
```

---

## 依存関係図

### Runtime依存関係

```
mathEntryInteraction.js (Main)
├── qtiCustomInteractionContext (TAO Core)
├── jquery_2_1_1 (TAO Portable Lib) ───┐
├── lodash (TAO Portable Lib) ─────────┤
├── mathquill/mathquill.js             │
│   ├── jquery ←───────────────────────┘
│   └── es6-collections (polyfill)
├── mathml-to-latex/mathml-to-latex.js
├── helper/ambiguousSymbols.js
└── helper/mathInPrompt.js
    ├── mathml-to-latex ↑
    └── mathquill ↑
```

### Creator依存関係

```
imsPciCreator.js (Entry Point)
├── creator/widget/Widget.js
│   └── creator/widget/states/states.js
│       ├── states/Question.js
│       │   ├── tpl/propertiesForm.tpl
│       │   ├── tpl/addGapBtn.tpl
│       │   └── helper/mathInPrompt.js
│       ├── states/Correct.js
│       │   └── tpl/answerForm.tpl
│       └── states/Map.js
│           ├── tpl/responseForm.tpl
│           ├── tpl/scoreForm.tpl
│           ├── tpl/answerForm.tpl
│           ├── tpl/alternativeForm.tpl
│           └── tpl/addAlternativeBtn.tpl
└── tpl/markup.tpl
```

---

## ファイル重要度ランキング

### ★★★ 最重要（コアロジック）

1. **`runtime/mathEntryInteraction.js`** (1,259行)
   - メインロジック全て
   - IMS PCI インターフェース実装
   - MathQuill管理
   - ツールバー生成
   - Gap Expression
   - レスポンス処理

2. **`creator/widget/states/Question.js`** (500行)
   - プロパティ設定
   - 80+ ツールボタン管理
   - Gap Expression切り替え

3. **`creator/widget/states/Map.js`** (700行)
   - 採点設定
   - 別解管理
   - Response Processing

### ★★ 重要（機能実装）

4. **`runtime/mathquill/mathquill.js`** (5,500行)
   - 数式エディタ
   - LaTeX変換
   - UI処理

5. **`creator/widget/states/Correct.js`** (100行)
   - 正答設定

6. **`creator/tpl/propertiesForm.tpl`**
   - プロパティUIの定義

### ★ 補助（ヘルパー・設定）

7. **`runtime/helper/ambiguousSymbols.js`** (56行)
   - 記号正規化

8. **`runtime/helper/mathInPrompt.js`** (61行)
   - プロンプト数式レンダリング

9. **`imsPciCreator.json`**
   - メタデータ

10. **各種テンプレート** (.tpl ファイル)
    - UI定義

11. **`rollup.config.js`**
    - ビルド設定

---

## まとめ

### アーキテクチャのポイント

1. **Runtime/Creator分離**:
   - Runtime: 軽量（受験者用）
   - Creator: 重量（編集機能全て）

2. **状態管理**:
   - Question: プロパティ設定
   - Correct: 正答設定
   - Map: 採点設定

3. **MathQuill依存**:
   - 数式入力UIの中核
   - 8年前のライブラリ（更新なし）

4. **AMD/Rollupビルド**:
   - TAOのAMDシステムに適合
   - jQuery/Lodashは外部化（TAO Coreから）

5. **Gap Expression**:
   - 空欄式問題のサポート
   - 動的フィールド追加・削除

### 改造時の注意点

1. **`mathEntryInteraction.js` がメイン**:
   - ほとんどのロジックがここに集中
   - 1,000行超の大規模ファイル

2. **MathQuill置き換え困難**:
   - 全体に深く統合されている
   - 代替には大規模リファクタリング必要

3. **ツールボタンカスタマイズ**:
   - `Question.js` の `tools` オブジェクト編集
   - `propertiesForm.tpl` のUI編集

4. **Gap機能拡張**:
   - `mathEntryInteraction.js` の Gap関連メソッド
   - `Question.js` の Gap切り替えロジック

5. **ビルド必須**:
   - ソース変更後は `npm run build` 必須
   - minファイルが実際に使用される

---

## 次のステップ

このドキュメントで各ファイルの役割を理解したら:

1. **設計ドキュメント**を参照: `docs/design/ja/` ディレクトリ
2. **セキュリティドキュメント**を参照: `docs/security/` ディレクトリ
3. **実際のコード**を読む: 重要度の高いファイルから順に
4. **改造計画**を立てる: 影響範囲を理解してから実装

---

**最終更新**: 2026-02-02
**バージョン**: 2.8.0
