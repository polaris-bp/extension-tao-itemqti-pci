# Math Entry Interaction PCI - Runtime モジュール仕様書

## 文書情報

| 項目 | 内容 |
|------|------|
| 作成日 | 2026-01-11 |
| バージョン | 2.8.0 |
| 対象モジュール | runtime/mathEntryInteraction.js |
| 文書種別 | モジュール仕様書 |

## 1. モジュール概要

### 1.1 目的

Runtime モジュールは、受験者がテストを受ける際に使用される数式入力インタラクションの実行時動作を提供する。

### 1.2 ファイル構成

```
runtime/
├── mathEntryInteraction.js       # メインモジュール
├── mathquill/
│   ├── mathquill.js              # MathQuill ライブラリ
│   ├── mathquill.css             # MathQuill スタイル
│   └── font/                     # Symbolaフォント
├── mathml-to-latex/
│   └── mathml-to-latex.js        # MathML変換ライブラリ
├── helper/
│   ├── mathInPrompt.js           # プロンプト内数式処理
│   └── ambiguousSymbols.js       # 曖昧な記号の正規化
├── polyfill/
│   └── es6-collections.js        # ES6 Map/Set ポリフィル
├── css/
│   └── mathEntryInteraction.css  # PCI スタイル
└── scss/
    └── mathEntryInteraction.scss # SCSS ソース
```

## 2. 主要ファイル仕様

### 2.1 mathEntryInteraction.js

#### 2.1.1 モジュール依存関係

**AMD依存：**
```javascript
define([
    'qtiCustomInteractionContext',      // PCI登録API
    'taoQtiItem/portableLib/jquery_2_1_1',  // jQuery
    'taoQtiItem/portableLib/lodash',    // Lodash
    'taoQtiItem/portableLib/OAT/util/event', // イベントマネージャ
    'mathEntryInteraction/runtime/mathquill/mathquill', // MathQuill
    'mathEntryInteraction/runtime/helper/mathInPrompt',
    'mathEntryInteraction/runtime/helper/ambiguousSymbols',
    'mathEntryInteraction/runtime/polyfill/es6-collections',
    'css!mathEntryInteraction/runtime/mathquill/mathquill',
    'css!mathEntryInteraction/runtime/css/mathEntryInteraction'
], function(...) { ... });
```

#### 2.1.2 グローバル変数

| 変数名 | 型 | 説明 |
|--------|---|------|
| `ns` | String | 名前空間（`.mathEntryInteraction`） |
| `cssClass` | Object | CSSクラス名定義 |
| `cssSelectors` | Object | CSSセレクタ定義 |
| `reSpace` | RegExp | 空白文字判定用正規表現 |
| `MQ` | Object | MathQuill API v2 インターフェース |
| `uidCounter` | Number | UID生成用カウンター |
| `labels` | Object | ロケール別ラベル定義 |

**cssClass定義：**
```javascript
var cssClass = {
    root: 'mq-root-block',      // MathQuillルート要素
    cursor: 'mq-cursor',        // カーソル要素
    newLine: 'mq-tao-br',       // 改行要素
    autoWrap: 'mq-tao-wrap'     // 自動折り返し要素
};
```

#### 2.1.3 ユーティリティ関数

##### htmlMarkup(cls, tag)

**目的：** シンプルなHTML要素の生成

**パラメータ：**
- `cls` (String): CSSクラス名
- `tag` (String, optional): タグ名（デフォルト: 'div'）

**戻り値：** HTML文字列

**実装例：**
```javascript
function htmlMarkup(cls, tag) {
    tag = tag || 'div';
    return `<${tag} class="${cls}"></${tag}>`;
}
```

##### getWidth(element)

**目的：** 要素の実際の幅を取得（margin含む）

**パラメータ：**
- `element` (DOMElement): 対象要素

**戻り値：** 幅（Number）

**計算式：**
```
width = rect.width + marginLeft + marginRight
        + (borderBox ? 0 : padding + border)
```

**特徴：**
- `getBoundingClientRect()` を使用（小数点以下も取得）
- box-sizing を考慮
- jQueryの `.width()` よりも正確

##### uid()

**目的：** 一意なID生成

**戻り値：** `"answer-{counter}"` 形式の文字列

**実装：**
```javascript
let uidCounter = 0;
const uid = () => `answer-${uidCounter++}`;
```

### 2.2 responsesManagerFactory()

#### 2.2.1 目的

複数の回答入力フィールドを管理するためのMapオブジェクトを拡張したマネージャーを生成。

#### 2.2.2 データ構造

```javascript
Map {
    'answer-0': {
        input: HTMLElement,     // 入力フィールドDOM要素
        response: {             // 回答データ
            base: {
                string: 'x^2+1'  // LaTeX文字列
            }
        }
    },
    'answer-1': { ... }
}
```

#### 2.2.3 拡張メソッド

| メソッド | パラメータ | 戻り値 | 説明 |
|---------|-----------|--------|------|
| `getFirstItem(index)` | index (optional) | Object | 最初のアイテムまたは指定インデックスのアイテムを取得 |
| `getIndex(index)` | index (optional) | String | 有効なインデックスを取得 |
| `currentIndex(index)` | index (optional) | String/undefined | 現在のインデックスを取得/設定 |

**実装：**
```javascript
function responsesManagerFactory() {
    const list = new Map();
    let currentIndex = null;

    Object.assign(list, {
        getFirstItem(index) {
            return list.get(this.getIndex(index));
        },
        getIndex(index) {
            if (typeof index === 'undefined') {
                const [inputIndex] = list.keys();
                return inputIndex;
            }
            return index;
        },
        currentIndex(index) {
            if (typeof index !== 'undefined') {
                currentIndex = index;
                return;
            }
            return currentIndex;
        }
    });

    return list;
}
```

## 3. mathEntryInteractionFactory 仕様

### 3.1 オブジェクト構造

mathEntryInteractionFactoryは、以下のプロパティとメソッドを持つオブジェクトを返す。

#### 3.1.1 プロパティ

| プロパティ | 型 | 説明 |
|-----------|---|------|
| `dom` | DOMElement | PCIのルートDOM要素 |
| `$container` | jQuery | PCIコンテナのjQueryオブジェクト |
| `$toolbar` | jQuery | ツールバー要素 |
| `$inputPlaceholder` | jQuery | プレースホルダー要素 |
| `inputs` | Map | responsesManager インスタンス |
| `mathField` | MathQuill | MathQuillインスタンス |
| `config` | Object | PCI設定オブジェクト |
| `userLanguage` | String | ユーザー言語コード |
| `pciInstance` | Object | PCIインスタンス（外部参照用） |
| `wrapCache` | WeakMap | 自動折り返しキャッシュ |
| `_inQtiCreator` | Boolean | Creatorコンテキストフラグ |
| `_activeGapFieldIndex` | Number | アクティブなGapフィールドのインデックス |

### 3.2 初期化メソッド

#### 3.2.1 initialize(dom, config, responsesManager)

**目的：** PCIの初期化

**パラメータ：**
- `dom` (DOMElement): PCIコンテナ
- `config` (Object): 設定オブジェクト
- `responsesManager` (Map): レスポンスマネージャ

**処理フロー：**
```
1. DOMとコンテナを保存
2. ユーザー言語を設定
3. ツールバー要素を取得
4. 入力フィールドを responsesManager に登録
5. render(config) を呼び出し
```

**実装詳細：**
```javascript
initialize: function initialize(dom, config, responsesManager) {
    this.dom = dom;
    this.userLanguage = config.userLanguage ?
        config.userLanguage.replace(/[-_][A-Z].*$/i, '').toLowerCase() : '';

    this.$container = $(dom);
    this.$toolbar = this.$container.find('.toolbar');

    const $input = this.$container.find('.math-entry-input');
    const id = uid();
    this.inputs = responsesManager;

    let responseValue = {};
    if (this.inputs.has(id) &&
        Object.keys(this.inputs.get(id)).includes('response')) {
        responseValue = { response: this.inputs.get(id).response };
    }

    this.inputs.set(id, Object.assign(responseValue, {input: $input[0]}));
    this.render(config);
}
```

#### 3.2.2 initConfig(config)

**目的：** 設定オブジェクトの初期化と正規化

**パラメータ：**
- `config` (Object): 生の設定オブジェクト

**設定項目：**

| プロパティ | 型 | デフォルト | 説明 |
|-----------|---|-----------|------|
| `isReviewMode` | Boolean | false | レビューモードフラグ |
| `authorizeWhiteSpace` | Boolean | false | スペースキー許可 |
| `focusOnDenominator` | Boolean | false (ja: true) | 分母へのフォーカス |
| `useGapExpression` | Boolean | false | Gap Expressionモード |
| `inResponseState` | Boolean | false | 正解入力状態フラグ |
| `gapExpression` | String | '' | Gap式のLaTeX |
| `gapStyle` | String | - | Gapスタイル（small/medium/large） |
| `locale` | String | 'en' | ロケール |
| `toolsStatus` | Object | {...} | 各ツールの有効/無効 |
| `allowNewLine` | Boolean | false | 改行許可（実験的） |
| `enableAutoWrap` | Boolean | false | 自動折り返し（実験的） |

**toBoolean ヘルパー：**
```javascript
function toBoolean(value, defaultValue) {
    if (typeof(value) === "undefined") {
        return defaultValue;
    } else {
        return (value === true || value === "true");
    }
}
```

**tool_{name} プロパティ：**

全てのツールバーツールについて `tool_{toolId}` プロパティが存在する。例：
- `tool_frac` (分数)
- `tool_sqrt` (平方根)
- `tool_exp` (指数)
- `tool_pi` (π)
- etc.

### 3.3 レンダリングメソッド

#### 3.3.1 render(config)

**目的：** PCIのUIをレンダリング

**処理フロー：**

```
1. initConfig(config)
2. createToolbar()
3. togglePlaceholder(false)
4. toggleResponseCorrectRow(false)
5. [モード判定による分岐]
6. 各モードに応じたレンダリング処理
```

**モード別処理：**

| 条件 | 処理内容 |
|------|---------|
| Gap + Creator + Response | `createMathStatic()` + Gap編集可能 |
| Gap + Creator + Question | `createMathEditable(true)` + Gap挿入可能 |
| Normal + Creator + Question | `createMathStatic()` + Placeholder |
| Normal + Creator + Response | `createMathEditable(true)` + 正解入力 |
| Review + Normal | `createMathStatic()` 読み取り専用 |
| Review + Gap | `createMathStatic()` + `setGapsDisabled(true)` |
| Gap + Runtime | `createMathStatic()` + Gap編集可能 |
| Normal + Runtime | `createMathEditable(false)` 通常入力 |

**実装例：**
```javascript
render: function render(config) {
    this.initConfig(config);
    this.createToolbar();
    this.togglePlaceholder(false);
    this.toggleResponseCorrectRow(false);

    if (this.inGapMode() && this.inQtiCreator() && this.inResponseState()) {
        this.setMathStaticContent(this.config.gapExpression);
        this.createMathStatic();
        this.monitorActiveGapField();
        this.removeSelectedInput();
        this.addToolbarListeners();
        this.addGapStyle();
        this.autoWrapContent();
        this.toggleResponseCorrectRow(true);
    }
    // ... 他のモード
}
```

#### 3.3.2 postRender()

**目的：** レンダリング後処理（プロンプト内数式の処理）

**処理：**
```javascript
postRender: function postRender() {
    const $prompt = this.$container.find('.prompt');
    mathInPrompt.postRender($prompt);
}
```

### 3.4 MathQuill管理メソッド

#### 3.4.1 getMqConfig()

**目的：** MathQuill設定オブジェクトの生成

**戻り値：** MathQuill設定オブジェクト

**主要設定：**

| 設定項目 | 値 | 説明 |
|---------|---|------|
| `spaceBehavesLikeTab` | `!authorizeWhiteSpace` | スペースキーの動作 |
| `focusOnDenominator` | `config.focusOnDenominator` | 分数入力時の動作 |
| `handlers.edit` | Function | 編集時のコールバック |
| `handlers.enter` | Function | Enterキー押下時のコールバック |

**edit ハンドラ：**
```javascript
edit: function onChange(mathField) {
    self.autoWrapContent();
    if (self.pciInstance) {
        let index = null;
        const $mathFieldInput = $(mathField.__controller.container[0]);
        index = $mathFieldInput.data('index') ||
                $mathFieldInput.parents('.math-entry-input').data('index');
        self.pciInstance.trigger('responseChange', [mathField.latex(), index]);
    }
}
```

**enter ハンドラ：**
```javascript
enter: function onEnter(mathField) {
    function isBrAllowed() {
        return self.config.allowNewLine &&
            ((!self.inGapMode() && !self.inQtiCreator()) ||
             (self.inGapMode() && self.inQtiCreator()));
    }

    if (isBrAllowed()) {
        mathField.write('\\embed{br}');
    }
}
```

#### 3.4.2 createMathEditable(replaceStatic, index)

**目的：** 編集可能なMathQuillフィールドを作成

**パラメータ：**
- `replaceStatic` (Boolean): 既存の静的フィールドを置き換えるか
- `index` (String, optional): 入力フィールドのインデックス

**処理ロジック：**
```
IF mathField が存在 AND replaceStatic === false THEN
    config を更新
ELSE IF mathField が存在 AND replaceStatic === true THEN
    要素をクリア
    新しい MathField を作成
ELSE
    新しい MathField を作成
END IF
```

**実装：**
```javascript
createMathEditable: function createMathEditable(replaceStatic, index) {
    const config = this.getMqConfig();
    const item = this.inputs.getFirstItem(index);

    if (this.mathField && this.mathField instanceof MathQuill && replaceStatic === false) {
        this.mathField.config(config);
    } else if (this.mathField && this.mathField instanceof MathQuill && !replaceStatic) {
        $(item.input).empty();
        this.mathField = MQ.MathField(item.input, config);
    } else {
        this.mathField = MQ.MathField(item.input, config);
    }
}
```

#### 3.4.3 createMathStatic(index)

**目的：** 静的数式フィールドを作成（Gap用）

**パラメータ：**
- `index` (Number, optional): 入力フィールドのインデックス

**処理：**
```javascript
createMathStatic: function createMathStatic(index) {
    const item = this.inputs.getFirstItem(index);
    this.mathField = MQ.StaticMath(item.input);

    const gapFields = this.getGapFields();
    gapFields.forEach(field => {
        field.config(this.getMqConfig());
    });
}
```

**注意：** `MQ.StaticMath()` は、`\\MathQuillMathField{}` タグをGapとして認識する。

#### 3.4.4 setMathStaticContent(latex, index)

**目的：** 静的数式のコンテンツを設定（Gap用）

**パラメータ：**
- `latex` (String): LaTeX文字列
- `index` (String, optional): 入力フィールドのインデックス

**変換処理：**
```javascript
latex = latex
    .replace(/\\taoGap/g, '\\MathQuillMathField{}')  // Gap → MQフィールド
    .replace(/\\taoBr/g, '\\embed{br}');             // 改行
$(item.input).text(latex);
```

#### 3.4.5 setLatex(latex, indexInput)

**目的：** LaTeX文字列を数式フィールドに設定

**パラメータ：**
- `latex` (String | String[]): LaTeX文字列（Gapモードでは配列）
- `indexInput` (String, optional): 入力フィールドのインデックス

**Gapモード処理：**
```javascript
if (this.inGapMode() && _.isArray(latex)) {
    const gapFields = this.getGapFields();
    latex.forEach(function (latexExpr, i) {
        if (gapFields[i]) {
            gapFields[i].latex(latexExpr);
        }
    });
}
```

**通常モード処理：**
```javascript
else {
    latex = latex
        .replace(/\\taoGap/g, '\\embed{gap}')
        .replace(/\\taoBr/g, '\\embed{br}')
        .replace(/\\text\{\}/g, '\\text{ }');

    if (!this.mathField) {
        const item = this.inputs.getFirstItem(indexInput);
        const config = this.getMqConfig();
        this.mathField = MQ.MathField(item.input, config);
    }
    this.mathField.latex(latex);
}
```

#### 3.4.6 insertLatex(latex, fn)

**目的：** カーソル位置にLaTeXを挿入

**パラメータ：**
- `latex` (String): 挿入するLaTeX
- `fn` (String): 'cmd' または 'write'

**挿入方法の違い：**

| fn値 | 動作 | 用途 | 例 |
|------|------|------|---|
| `cmd` | コマンドとして挿入、カーソルは内部へ | 構造的な要素 | `\sqrt`, `\frac` |
| `write` | テキストとして挿入、カーソルは右へ | 記号・テキスト | `+`, `-`, `\pi` |

**実装：**
```javascript
insertLatex: function insertLatex(latex, fn) {
    const activeMathField = this.getActiveMathField();

    if (activeMathField && _.isFunction(activeMathField[fn])) {
        activeMathField[fn](latex);
        activeMathField.focus();
    }
}
```

#### 3.4.7 getActiveMathField()

**目的：** 現在アクティブなMathQuillフィールドを取得

**戻り値：** MathQuillインスタンス

**ロジック：**
```
IF Gapモード AND (Responseモード OR Runtime) THEN
    activeGapFieldIndex をデフォルト化
    gapFields[activeGapFieldIndex] を返す
ELSE
    this.mathField を返す
END IF
```

**実装：**
```javascript
getActiveMathField: function getActiveMathField() {
    let activeMathField = null;

    if ((this.inGapMode() && this.inResponseState()) ||
        (this.inGapMode() && !this.inQtiCreator())) {

        if (!_.isFinite(this._activeGapFieldIndex)) {
            this._activeGapFieldIndex = 0;
        }

        const gapFields = this.getGapFields();
        if (gapFields.length > 0) {
            activeMathField = gapFields[this._activeGapFieldIndex];
        }
    } else {
        activeMathField = this.mathField;
    }
    return activeMathField;
}
```

#### 3.4.8 getGapFields()

**目的：** 全てのGapフィールドを取得

**戻り値：** MathQuillインスタンスの配列

**実装：**
```javascript
getGapFields: function getGapFields() {
    return (this.mathField && _.isArray(this.mathField.innerFields))
        ? this.mathField.innerFields
        : [];
}
```

**注意：** `mathField.innerFields` は、`StaticMath` のGapフィールドを保持する配列。

### 3.5 Gapモード専用メソッド

#### 3.5.1 monitorActiveGapField(inputIndex)

**目的：** アクティブなGapフィールドを追跡

**処理：**
```javascript
monitorActiveGapField: function monitorActiveGapField(inputIndex) {
    if (!inputIndex) {
        const [index] = this.inputs.keys();
        inputIndex = index;
    }

    const $editableFields = $(this.inputs.get(inputIndex).input)
        .find('.mq-editable-field');

    this._activeGapFieldIndex = null;

    if ($editableFields.length) {
        $.each($editableFields, (fieldIndex, input) => {
            $(input)
                .off(ns)
                .on(`click${ns} keyup${ns}`, () => {
                    this._activeGapFieldIndex = fieldIndex;
                });
        });
    }
}
```

**イベント：**
- `click.mathEntryInteraction`
- `keyup.mathEntryInteraction`

#### 3.5.2 addGap()

**目的：** Gapを追加（Creator専用）

**実装：**
```javascript
addGap: function addGap() {
    if (this.inQtiCreator()) {
        this.insertLatex('\\embed{gap}', 'write');
    }
}
```

#### 3.5.3 setGapsDisabled(disabled)

**目的：** Gapフィールドの有効/無効を切り替え

**パラメータ：**
- `disabled` (Boolean): 無効化フラグ（デフォルト: true）

**実装：**
```javascript
setGapsDisabled: function setGapsDisabled(disabled = true) {
    this.getGapFields().forEach(gapField => {
        const textarea = gapField.el().querySelector('textarea');
        if (textarea) {
            textarea.disabled = !!disabled;
        }
    });
}
```

#### 3.5.4 hideCursors()

**目的：** 無効なGapのカーソルを非表示

**実装：**
```javascript
hideCursors: function hideCursors() {
    this.$container.find('.mq-editable-field').addClass('hidden-cursor');
}
```

#### 3.5.5 addGapStyle()

**目的：** Gapスタイルクラスの適用

**処理：**
```javascript
addGapStyle: function addGapStyle() {
    if (this.config.gapStyle) {
        // 既存のmath-gap-*クラスを削除
        this.$container.removeClass(function (index, className) {
            return (className.match(/\bmath-gap-[\w]+\b/g) || []).join(' ');
        });

        // 新しいスタイルを追加
        this.$container.addClass(this.config.gapStyle);
    }

    // 代替回答のwrapを表示
    const inputWrap = this.$container.find('.math-entry-response-wrap');
    if (inputWrap.length > 0) {
        $(inputWrap[0]).show();
    }
}
```

**Gapスタイルクラス：**
- `math-gap-small` (小)
- `math-gap-medium` (中)
- `math-gap-large` (大)

### 3.6 ツールバーメソッド

#### 3.6.1 createToolbar()

**目的：** ツールバーUIを生成

**処理フロー：**
```
1. availableTools 定義（全ツール）
2. availableToolGroups 定義（グループ化）
3. $toolbar をクリア
4. 各ツールグループを生成して追加
5. 日本語ロケールの場合、分数ツールのスタイル調整
```

**ツール定義例：**
```javascript
var availableTools = {
    frac: {
        label: self.getLabel('x/y'),
        latex: '\\frac',
        fn: 'cmd',
        desc: 'Fraction'
    },
    sqrt: {
        label: '<svg>...</svg>',
        latex: '\\sqrt',
        fn: 'cmd',
        desc: 'Square root'
    },
    // ... 他のツール
};
```

**ツールグループ定義：**
```javascript
var availableToolGroups = [
    {id: 'functions', tools: ['sqrt', 'frac', 'exp', ...]},
    {id: 'symbols', tools: ['e', 'infinity', 'lparen', ...]},
    {id: 'geometry', tools: ['angle', 'triangle', ...]},
    {id: 'trigo', tools: ['pi', 'sin', 'cos', 'tan']},
    {id: 'comparison', tools: ['lower', 'greater', ...]},
    {id: 'operands', tools: ['equal', 'plus', 'minus', ...]},
    {id: 'matrix', tools: ['matrix_2row', 'matrix_2row_2col']}
];
```

#### 3.6.2 createToolGroup(group, availableTools)

**目的：** ツールグループのUIを生成

**パラメータ：**
- `group` (Object): グループ定義 `{id, tools}`
- `availableTools` (Object): 利用可能なツール定義

**戻り値：** jQuery要素 または 空文字列

**実装：**
```javascript
createToolGroup: function createToolGroup(group, availableTools) {
    var self = this,
        $toolGroup = $('<div>', {
            'class': 'math-entry-toolgroup',
            'data-identifier': group.id
        }),
        activeTools = 0;

    group.tools.forEach(function (toolId) {
        var toolConfig = availableTools[toolId];
        toolConfig.id = toolId;

        if (self.config.toolsStatus[toolId] === true) {
            $toolGroup.append(self.createTool(toolConfig));
            activeTools++;
        }
    });

    return (activeTools > 0) ? $toolGroup : '';
}
```

#### 3.6.3 createTool(config)

**目的：** 個別ツールボタンの生成

**パラメータ：**
- `config` (Object): ツール設定 `{id, latex, fn, label}`

**戻り値：** jQuery要素

**実装：**
```javascript
createTool: function createTool(config) {
    return $('<div>', {
        'class': 'math-entry-tool',
        'data-identifier': config.id,
        'data-latex': config.latex,
        'data-fn': config.fn,
        html: config.label
    });
}
```

#### 3.6.4 addToolbarListeners()

**目的：** ツールバーイベントリスナーの登録

**イベント：** `mousedown.mathEntryInteraction`

**実装：**
```javascript
addToolbarListeners: function addToolbarListeners() {
    var self = this;
    this.$toolbar
        .off(`mousedown${ns}`)
        .on(`mousedown${ns}`, function (e) {
            var $target, fn = '', latex = '';

            if ($(e.target).data('fn')) {
                $target = $(e.target);
            } else {
                $target = $(e.target.parentElement);
            }

            fn = $target.data('fn');
            latex = $target.data('latex');

            e.stopPropagation();
            e.preventDefault();

            self.insertLatex(latex, fn);
        });
}
```

### 3.7 レスポンス管理メソッド

#### 3.7.1 getResponse(inputId)

**目的：** 受験者の回答を取得（IMS PCI標準）

**パラメータ：**
- `inputId` (String, optional): 入力フィールドID

**戻り値：** レスポンスオブジェクト
```javascript
{
    base: {
        string: 'latex...'
    }
}
```

**Gapモード処理：**
```javascript
if (this.inGapMode()) {
    response = {
        base: {
            string: this.getGapFields()
                .map(function (gapField) {
                    return convertAmbiguousSymbols(gapField.latex());
                }).toString()  // カンマ区切り
        }
    };
}
```

**通常モード処理：**
```javascript
else {
    response = {
        base: {
            string: convertAmbiguousSymbols(this.mathField.latex())
        }
    };
}
```

**空チェック：**
```javascript
return response.base.string.replace(/,/g, '') !== ''
    ? response
    : {base: {string: ''}};
```

#### 3.7.2 setResponse(response)

**目的：** 受験者の回答を設定（IMS PCI標準）

**パラメータ：**
- `response` (Object): レスポンスオブジェクト

**Gapモード処理：**
```javascript
if (this.inGapMode()) {
    if (response && response.base && response.base.string) {
        const gapFields = this.getGapFields();
        const gaps = response.base.string.split(',');
        gapFields.forEach(function (gap, index) {
            gap.latex(gaps[index]);
        });
    }
}
```

**通常モード処理：**
```javascript
else {
    if (response && response.base && response.base.string) {
        this.setLatex(response.base.string);
    }
}
```

#### 3.7.3 resetResponse()

**目的：** 回答のリセット

**処理：**
```javascript
resetResponse: function resetResponse() {
    const gapFields = this.getGapFields();
    if (this.inGapMode()) {
        gapFields.forEach(function (gapField) {
            gapField.latex('');
        });
    } else {
        this.setLatex('');
    }
}
```

### 3.8 状態管理メソッド

#### 3.8.1 getSerializedState()

**目的：** 状態をシリアライズ（IMS PCI標準）

**戻り値：**
```javascript
{
    response: {
        base: {
            string: 'latex...'
        }
    }
}
```

**実装：**
```javascript
getSerializedState: function getSerializedState() {
    return {response: this.getResponse()};
}
```

#### 3.8.2 setSerializedState(state)

**目的：** シリアライズされた状態を復元（IMS PCI標準）

**パラメータ：**
- `state` (Object): 状態オブジェクト

**実装：**
```javascript
setSerializedState: function setSerializedState(state) {
    if (state && state.response) {
        this.setResponse(state.response);
    }
}
```

### 3.9 クリーンアップメソッド

#### 3.9.1 destroy()

**目的：** リソースのクリーンアップ（IMS PCI標準）

**処理：**
```javascript
destroy: function destroy() {
    // イベントリスナー削除
    for (let value of this.inputs.values()) {
        $(value.input).find('.mq-editable-field').off(ns);
        $(value.input).off(ns);
    }
    this.$toolbar.off(ns);

    // レスポンスリセット
    this.resetResponse();

    // MathQuill復元
    if (this.mathField instanceof MathQuill) {
        this.mathField.revert();
    }
}
```

### 3.10 ヘルパーメソッド

#### 3.10.1 inQtiCreator()

**目的：** Creatorコンテキストかどうかを判定

**戻り値：** Boolean

**実装：**
```javascript
inQtiCreator: function isInCreator() {
    if (_.isUndefined(this._inQtiCreator) && this.$container) {
        this._inQtiCreator = this.$container.hasClass('tao-qti-creator-context') ||
            this.$container.find('.tao-qti-creator-context').length > 0;
    }
    return this._inQtiCreator;
}
```

#### 3.10.2 inGapMode()

**目的：** Gap Expressionモードかどうかを判定

**戻り値：** Boolean

**実装：**
```javascript
inGapMode: function inGapMode() {
    return this.config.useGapExpression;
}
```

#### 3.10.3 inResponseState()

**目的：** 正解入力状態かどうかを判定

**戻り値：** Boolean

**実装：**
```javascript
inResponseState: function inResponseState() {
    return this.config.inResponseState;
}
```

#### 3.10.4 inJapanese()

**目的：** 日本語ロケールかどうかを判定

**戻り値：** Boolean

**実装：**
```javascript
inJapanese: function inJapanese() {
    return this.userLanguage === 'ja';
}
```

#### 3.10.5 getLabel(label)

**目的：** ロケール別ラベルを取得

**パラメータ：**
- `label` (String): ラベルキー

**戻り値：** ローカライズされたラベル

**実装：**
```javascript
getLabel: function getLabel(label) {
    var localizedLabels = labels[this.config.locale];
    if (localizedLabels) {
        return localizedLabels[label] || label;
    }
    return label;
}
```

**日本語ラベル定義：**
```javascript
const labels = {
    'ja': {
        'x/y': '<span>x</span><br><span>y</span>',  // 縦分数
        '&le;': '&#8806;',  // ≦
        '&ge;': '&#8807;',  // ≧
        '\\le': '\\leq',
        '\\ge': '\\geq'
    }
};
```

### 3.11 実験的機能メソッド

#### 3.11.1 autoWrapContent()

**目的：** コンテンツの自動折り返し処理

**条件：** `config.enableAutoWrap === true`

**アルゴリズム：**
```
1. コンテナ幅を取得
2. WeakMap キャッシュを初期化
3. 全子ノードを走査
4. 各ブロックの幅を計算
5. 行幅が maxWidth を超えたら改行挿入
6. 最後のスペース位置を記憶して最適な位置で改行
```

**実装の要点：**
```javascript
autoWrapContent: function autoWrapContent() {
    if (this.config.enableAutoWrap) {
        const [inputIndex] = this.inputs.keys();
        $container = $(this.inputs.get(inputIndex).input).find(cssSelectors.root);

        maxWidth = $container.width();

        if (!this.wrapCache) {
            this.wrapCache = new window.WeakMap();
        }
        cache = this.wrapCache;

        nodes = _.toArray($container.get(0).childNodes);
        for (length = nodes.length, index = 0; index < length; index++) {
            node = nodes[index];
            block = cache.get(node);

            // ブロック情報をキャッシュ
            if (!block) {
                block = {
                    classList: node.classList,
                    isSpace: reSpace.test(node.innerHTML)
                };
                cache.set(node, block);
            }

            // 幅計算と改行判定
            // ...
        }
    }
}
```

## 4. ヘルパーモジュール仕様

### 4.1 ambiguousSymbols.js

#### 4.1.1 目的

曖昧なUnicode記号をASCII相当に正規化する。

#### 4.1.2 変換マッピング

```javascript
const defaultMapping = {
    '０': '0', '１': '1', '２': '2', '３': '3', '４': '4',
    '５': '5', '６': '6', '７': '7', '８': '8', '９': '9',
    '−': '-',  // マイナス記号
    '‐': '-',  // ハイフン
    '―': '-',  // ダッシュ
    '-': '-'   // 全角ハイフン
};
```

#### 4.1.3 convert(text)

**パラメータ：**
- `text` (String): 変換対象テキスト

**戻り値：** 正規化されたテキスト

**実装：**
```javascript
function convert(text) {
    let result = '';
    for (const char of text) {
        result += defaultMapping[char] || char;
    }
    return result;
}
```

### 4.2 mathInPrompt.js

#### 4.2.1 目的

プロンプト内の数式を適切にレンダリングする。

#### 4.2.2 postRender($prompt)

**パラメータ：**
- `$prompt` (jQuery): プロンプト要素

**処理：**
- プロンプト内のLaTeXやMathMLを検出してレンダリング

（詳細実装は別ファイルにあるため、APIのみ記載）

## 5. MathQuillカスタムEmbed

### 5.1 Gap Embed

**登録：**
```javascript
MQ.registerEmbed('gap', function registerGap() {
    return {
        htmlString: htmlMarkup('mq-tao-gap', 'span'),
        text: function text() {
            return 'tao_gap';
        },
        latex: function latex() {
            return '\\taoGap';
        }
    };
});
```

**用途：** Gap Expressionモードでのプレースホルダー

**LaTeX記法：** `\embed{gap}`

### 5.2 Line Break Embed

**登録：**
```javascript
MQ.registerEmbed('br', function registerBr() {
    return {
        htmlString: htmlMarkup(cssClass.newLine),
        text: function text() {
            return 'tao_br';
        },
        latex: function latex() {
            return '\\taoBr';
        }
    };
});
```

**用途：** 手動改行の挿入

**LaTeX記法：** `\embed{br}`

## 6. PCI登録

### 6.1 qtiCustomInteractionContext.register()

**登録内容：**
```javascript
qtiCustomInteractionContext.register({
    typeIdentifier: 'mathEntryInteraction',

    getInstance: function getInstance(dom, config, state) {
        const responsesManager = responsesManagerFactory();
        const mathEntryInteraction = mathEntryInteractionFactory(responsesManager);

        const pciInstance = {
            getResponse: function() {
                return mathEntryInteraction.getResponse();
            },
            getState: function() {
                return mathEntryInteraction.getSerializedState();
            },
            oncompleted: function() {
                pciInstance.off('configChange');
                pciInstance.off('addGap');
                pciInstance.off('latexInput');
                pciInstance.off('latexGapInput');
                mathEntryInteraction.destroy();
            },
            getResponsesManager() {
                return responsesManager;
            }
        };

        event.addEventMgr(pciInstance);

        mathEntryInteraction.initialize(dom, config.properties, responsesManager);

        // イベントハンドラ登録
        // ...

        mathEntryInteraction.postRender();
        config.onready(pciInstance);
        mathEntryInteraction.pciInstance = pciInstance;
    }
});
```

### 6.2 イベントハンドラ

#### configChange
```javascript
pciInstance.on('configChange', function (properties) {
    mathEntryInteraction.render(properties);
    mathEntryInteraction.postRender();
});
```

#### latexInput
```javascript
pciInstance.on('latexInput', function (latex, indexInput) {
    if (!mathEntryInteraction.inputs.has(indexInput)) {
        return false;
    }
    mathEntryInteraction.mathField = MQ.MathField(
        mathEntryInteraction.inputs.get(indexInput).input,
        config
    );
    mathEntryInteraction.setLatex(latex, indexInput);
    mathEntryInteraction.mathField.focus();
});
```

#### latexGapInput
```javascript
pciInstance.on('latexGapInput', function (gapLatex, indexInput) {
    if (gapLatex.base && _.isArray(gapLatex.base.string)) {
        if (!mathEntryInteraction.inputs.has(indexInput)) {
            return false;
        }
        mathEntryInteraction.mathField = MQ.StaticMath(
            mathEntryInteraction.inputs.get(indexInput).input,
            config
        );
        const gaps = mathEntryInteraction.getGapFields();
        gaps.forEach(function (gap, index) {
            if (typeof gapLatex.base.string[index] !== 'undefined') {
                gap.latex(gapLatex.base.string[index]);
            }
        });
    } else {
        mathEntryInteraction.config.gapExpression = gapLatex;
    }
});
```

#### addGap
```javascript
pciInstance.on('addGap', function () {
    mathEntryInteraction.addGap();
});
```

#### addAlternative
```javascript
pciInstance.on('addAlternative', function (latex, gapValues, responseId) {
    mathEntryInteraction.addAlternative(latex, gapValues, responseId);
});
```

## 7. パフォーマンス考慮事項

### 7.1 最適化ポイント

| 項目 | 実装 | 効果 |
|------|------|------|
| DOM計測 | `getBoundingClientRect()` 使用 | jQueryより高速・正確 |
| キャッシング | WeakMap使用 | GC対象、メモリ効率的 |
| イベント委譲 | ツールバーで使用 | リスナー数削減 |
| 遅延評価 | `inQtiCreator()` のキャッシュ | 判定コストの削減 |

### 7.2 メモリ管理

- `destroy()` での確実なクリーンアップ
- WeakMap の活用（自動GC）
- イベントリスナーの名前空間化（`.mathEntryInteraction`）

## 変更履歴

| バージョン | 日付 | 変更内容 |
|-----------|------|---------|
| 2.8.0 | 2026-01-11 | 初版作成 |
