# Math Entry Interaction PCI - Creator モジュール仕様書

## 文書情報

| 項目 | 内容 |
|------|------|
| 作成日 | 2026-01-11 |
| バージョン | 2.8.0 |
| 対象モジュール | creator/ |
| 文書種別 | モジュール仕様書 |

## 1. モジュール概要

### 1.1 目的

Creator モジュールは、問題作成者がアイテムを編集する際に使用されるウィジェットと状態管理機能を提供する。

### 1.2 ファイル構成

```
creator/
├── widget/
│   ├── Widget.js              # メインウィジェット
│   └── states/
│       ├── states.js          # 状態定義
│       ├── Question.js        # 問題編集状態
│       ├── Correct.js         # 正解設定遷移状態
│       └── Map.js             # 正解・スコアリング設定状態
├── tpl/                       # Handlebarsテンプレート
│   ├── markup.tpl             # 基本マークアップ
│   ├── propertiesForm.tpl     # プロパティフォーム
│   ├── addGapBtn.tpl          # Gap追加ボタン
│   ├── addAlternativeBtn.tpl  # 代替回答追加ボタン
│   ├── alternativeForm.tpl    # 代替回答フォーム
│   ├── answerForm.tpl         # 回答フォーム
│   ├── responseForm.tpl       # レスポンスフォーム
│   └── scoreForm.tpl          # スコアフォーム
└── img/
    └── icon.svg               # PCIアイコン
```

## 2. Widget.js 仕様

### 2.1 概要

TAO QTI Creator フレームワークのカスタムインタラクションウィジェットを拡張したウィジェット。

### 2.2 依存関係

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/interactions/customInteraction/Widget',
    'mathEntryInteraction/creator/widget/states/states'
], function(Widget, states) {
    // ...
});
```

### 2.3 MathEntryInteractionWidget

**クラス構造：**
```javascript
var MathEntryInteractionWidget = Widget.clone();
```

**プロトタイプ拡張：**

#### initCreator()

**目的：** Creator初期化

**処理フロー：**
```
1. 状態を登録 (states)
2. 親クラスの initCreator を呼び出し
3. mathEntryInteraction 要素に 'tao-qti-creator-context' クラスを追加
```

**実装：**
```javascript
MathEntryInteractionWidget.initCreator = function initCreator() {
    var $interaction;

    this.registerStates(states);

    Widget.initCreator.call(this);

    $interaction = this.$container.find('.mathEntryInteraction');
    if ($interaction.length) {
        $interaction.addClass('tao-qti-creator-context');
    }
};
```

## 3. 状態管理仕様

### 3.1 states.js

#### 3.1.1 概要

TAO Creator のState Patternに基づく状態定義ファイル。

#### 3.1.2 状態定義

```javascript
define([
    'mathEntryInteraction/creator/widget/states/Question',
    'mathEntryInteraction/creator/widget/states/Correct',
    'mathEntryInteraction/creator/widget/states/Map'
], function(Question, Correct, Map){
    'use strict';

    return {
        Question : Question,
        Correct  : Correct,
        Map      : Map
    };
});
```

#### 3.1.3 状態遷移

```
Sleep (非アクティブ)
  ↓
Question (問題編集)
  ↓
Correct (遷移状態)
  ↓
Map (正解・スコアリング設定)
  ↓ (完了)
Question (に戻る)
```

## 4. Question State 仕様

### 4.1 概要

問題プロンプトの編集、プロパティ設定、ツールバーカスタマイズを担当する状態。

### 4.2 依存関係

```javascript
define([
    'jquery',
    'lodash',
    'i18n',
    'taoQtiItem/qtiCreator/widgets/states/factory',
    'taoQtiItem/qtiCreator/widgets/interactions/states/Question',
    'taoQtiItem/qtiCreator/widgets/helpers/formElement',
    'taoQtiItem/qtiCreator/editor/simpleContentEditableElement',
    'taoQtiItem/qtiCreator/editor/containerEditor',
    'tpl!mathEntryInteraction/creator/tpl/propertiesForm',
    'tpl!mathEntryInteraction/creator/tpl/addGapBtn',
    'mathEntryInteraction/runtime/helper/mathInPrompt'
], function($, _, __, stateFactory, Question, formElement, simpleEditor,
            containerEditor, formTpl, addGapBtnTpl, mathInPrompt) {
    // ...
});
```

### 4.3 ツール定義

#### 4.3.1 tools オブジェクト

各ツールは、依存するプロパティの配列を持つ。

**構造：**
```javascript
const tools = {
    tool_frac: {
        props: [{ name: 'tool_frac', defaultValue: true }]
    },
    squarebkts: {
        props: [
            { name: 'tool_rbrack', defaultValue: true },
            { name: 'tool_lbrack', defaultValue: true }
        ]
    },
    // ... 他のツール
};
```

**複合ツール：**
- `squarebkts`: 左右の角括弧
- `roundbkts`: 左右の丸括弧
- `curlybkts`: 左右の波括弧

**単一ツール：**
- `tool_frac`, `tool_sqrt`, `tool_exp`, `tool_log`, etc.

#### 4.3.2 ツールカテゴリ

| カテゴリ | ツールID |
|---------|---------|
| 関数 | frac, sqrt, exp, log, ln, limit, sum, nthroot, subscript |
| 記号 | e, infinity, pi, squarebkts, roundbkts, curlybkts, colon, to, degree, percent |
| 幾何 | angle, triangle, similar, paral, perp, vline |
| 三角関数 | sin, cos, tan |
| 比較 | lower, greater, lte, gte, approx |
| 演算子 | equal, plus, minus, times, timesdot, divide, plusminus |
| 集合 | inmem, ninmem, union, intersec, congruent, subset, superset, contains |
| 積分 | integral |
| 行列 | matrix_2row, matrix_2row_2col |

### 4.4 State Factory

#### 4.4.1 MathEntryInteractionStateQuestion

**基底クラス：** `taoQtiItem/qtiCreator/widgets/interactions/states/Question`

**ファクトリー構造：**
```javascript
var MathEntryInteractionStateQuestion = stateFactory.extend(
    Question,
    function create() {
        // 初期化処理
    },
    function exit() {
        // 終了処理
    }
);
```

### 4.5 初期化処理（create）

#### 4.5.1 create() メソッド

**処理フロー：**
```
1. コンテナ、プロンプト、interaction を取得
2. プロンプトエディタを作成（containerEditor）
3. Gap Expressionモードの場合、Add Gapボタンを作成
4. MathFieldリスナーを追加
```

**実装：**
```javascript
function create() {
    var $container = this.widget.$container,
        $prompt = $container.find('.prompt'),
        interaction = this.widget.element;

    // プロンプトエディタ作成
    containerEditor.create($prompt, {
        change: function(text) {
            interaction.data('prompt', text);
            interaction.updateMarkup();

            if (!$prompt.is('[data-html-editable-container="true"]')) {
                mathInPrompt.postRender($prompt);
            }
        },
        markup: interaction.markup,
        markupSelector: '.prompt',
        related: interaction,
        areaBroker: this.widget.getAreaBroker()
    });

    // Gap Expressionモードの処理
    if (toBoolean(interaction.prop('useGapExpression'), false)) {
        this.createAddGapBtn();
    }

    this.addMathFieldListener();
}
```

#### 4.5.2 プロンプトエディタ

**目的：** プロンプトテキストの編集機能を提供

**イベント：** `change` - プロンプト変更時

**処理：**
1. `interaction.data('prompt', text)` - データ更新
2. `interaction.updateMarkup()` - マークアップ更新
3. `mathInPrompt.postRender($prompt)` - 数式レンダリング

### 4.6 終了処理（exit）

#### 4.6.1 exit() メソッド

**処理：**
```javascript
function exit() {
    var $container = this.widget.$container,
        $prompt = $container.find('.prompt');

    simpleEditor.destroy($container);
    containerEditor.destroy($prompt);

    this.removeAddGapBtn();
}
```

**クリーンアップ対象：**
- simpleEditor
- containerEditor
- Add Gapボタン

### 4.7 プロパティフォーム

#### 4.7.1 initForm()

**目的：** プロパティフォームの初期化

**処理フロー：**
```
1. フォームテンプレートをレンダリング
2. フォームウィジェットを初期化
3. 変更コールバックを設定
4. Gap Style セレクターを初期化
5. グループチェックボックスの初期状態を設定
```

**実装：**
```javascript
MathEntryInteractionStateQuestion.prototype.initForm = function initForm() {
    var self = this,
        _widget = this.widget,
        $form = _widget.$form,
        interaction = _widget.element,
        response = interaction.getResponseDeclaration(),
        $gapStyleBox,
        $gapStyleSelector;

    // フォームレンダリング
    $form.html(formTpl(_.assign({
        serial: response.serial,
        identifier: interaction.attr('responseIdentifier'),
        authorizeWhiteSpace: toBoolean(interaction.prop('authorizeWhiteSpace'), false),
        useGapExpression: toBoolean(interaction.prop('useGapExpression'), false),
        focusOnDenominator: toBoolean(interaction.prop('focusOnDenominator'), false),
        allowNewLine: toBoolean(interaction.prop('allowNewLine'), false),
        enableAutoWrap: toBoolean(interaction.prop('enableAutoWrap'), false)
    }, getToolsInitValues(interaction))));

    // フォームウィジェット初期化
    formElement.initWidget($form);

    // 変更コールバック設定
    formElement.setChangeCallbacks($form, interaction, _.assign({
        identifier: function(i, value) {
            response.id(value);
            interaction.attr('responseIdentifier', value);
        },
        useGapExpression: function gapChangeCallback(i, value) {
            // Gap Expression切り替え処理
        },
        gapStyle: function gapStyleChangeCallback(i, newStyle) {
            // Gapスタイル変更処理
        },
        authorizeWhiteSpace: configChangeCallBack,
        focusOnDenominator: configChangeCallBack,
        toggle_all: toggleAllChangeCallBack,
        group: groupChangeCallBack,
        allowNewLine: configChangeCallBack,
        enableAutoWrap: configChangeCallBack
    }, getToolsCallBacks()));

    // Gap Styleセレクター初期化
    $gapStyleBox = $form.find('.mathgap-style-box');
    $gapStyleSelector = $gapStyleBox.find('[data-mathgap-style]');

    $gapStyleSelector.select2({
        width: '100%',
        minimumResultsForSearch: Infinity
    })
    .val(interaction.prop('gapStyle'))
    .trigger('change');

    // チェックボックスの初期状態設定
    setGroupCheckboxStates($form);
    setToggleAllCheckboxState($form);
};
```

### 4.8 変更コールバック

#### 4.8.1 configChangeCallBack(interaction, value, name)

**目的：** 汎用的なプロパティ変更ハンドラ

**パラメータ：**
- `interaction`: インタラクション要素
- `value`: 新しい値
- `name`: プロパティ名

**実装：**
```javascript
function configChangeCallBack(interaction, value, name) {
    interaction.prop(name, value);
    interaction.triggerPci('configChange', [interaction.getProperties()]);
}
```

#### 4.8.2 toggleAllChangeCallBack(interaction, value)

**目的：** 全ツールの一括切り替え

**処理：**
```javascript
function toggleAllChangeCallBack(interaction, value) {
    const $checkboxes = $(this).closest('.tools').find('[type="checkbox"]');

    toggleCheckBoxes(interaction, $checkboxes, value);
    interaction.triggerPci('configChange', [interaction.getProperties()]);
}
```

#### 4.8.3 groupChangeCallBack(interaction, value)

**目的：** ツールグループの一括切り替え

**処理：**
```javascript
function groupChangeCallBack(interaction, value) {
    const $this = $(this);
    const $checkboxes = $this.closest('.group').find('[type="checkbox"]');

    toggleCheckBoxes(interaction, $checkboxes, value);

    // 親チェックボックスの状態を更新
    const $allTools = $this.closest('.tools');
    updateParentCheckboxState($allTools, '[name="toggle_all"]');

    interaction.triggerPci('configChange', [interaction.getProperties()]);
}
```

#### 4.8.4 toolChangeCallBack(interaction, value, name)

**目的：** 個別ツールの切り替え

**処理：**
```javascript
function toolChangeCallBack(interaction, value, name) {
    if (tools[name]) {
        tools[name].props.forEach(prop => interaction.prop(prop.name, value));
    }

    const $this = $(this);
    const $group = $this.closest('.group');
    const $allTools = $this.closest('.tools');

    // グループチェックボックスの状態を更新
    updateParentCheckboxState($group, '[name="group"]');

    // toggle_allチェックボックスの状態を更新
    updateParentCheckboxState($allTools, '[name="toggle_all"]');

    interaction.triggerPci('configChange', [interaction.getProperties()]);
}
```

### 4.9 チェックボックス管理

#### 4.9.1 toggleCheckBoxes(interaction, $checkboxes, value)

**目的：** 複数のチェックボックスを一括切り替え

**処理：**
```javascript
function toggleCheckBoxes(interaction, $checkboxes, value) {
    $checkboxes.prop({ checked: value, indeterminate: false });

    $checkboxes.each(function() {
        const propName = $(this).attr('name');
        if (tools[propName]) {
            tools[propName].props.forEach(prop => {
                interaction.prop(prop.name, value);
            });
        }
    });
}
```

#### 4.9.2 updateParentCheckboxState($container, parentSelector)

**目的：** 親チェックボックスの状態を子に合わせて更新

**状態：**
- checked: 全て選択
- unchecked: 全て未選択
- indeterminate: 一部選択

**実装：**
```javascript
function updateParentCheckboxState($container, parentSelector) {
    const $parentCheckbox = $container.find(parentSelector);
    const $childCheckboxes = $container.find('[type="checkbox"]').not(parentSelector);

    const checkedCount = $childCheckboxes.filter(':checked').length;
    const allUnchecked = checkedCount === 0;
    const allChecked = checkedCount === $childCheckboxes.length;

    if ($parentCheckbox.length) {
        $parentCheckbox[0].checked = allChecked;
        $parentCheckbox[0].indeterminate = !allUnchecked && !allChecked;
    }
}
```

### 4.10 Gap機能

#### 4.10.1 createAddGapBtn()

**目的：** "Add Gap" ボタンの作成

**処理：**
```javascript
MathEntryInteractionStateQuestion.prototype.createAddGapBtn = function createAddGapBtn() {
    var _widget = this.widget,
        $container = _widget.$container,
        $toolbar = $container.find('.toolbar'),
        interaction = _widget.element;

    if ($toolbar.length) {
        $toolbar.after($addGapBtn);
        $addGapBtn.on('click', function() {
            interaction.getResponseDeclaration().removeMapEntries();
            interaction.triggerPci('addGap');
        });
    }
};
```

**イベント：**
- `click` → `removeMapEntries()` + `triggerPci('addGap')`

#### 4.10.2 removeAddGapBtn()

**目的：** "Add Gap" ボタンの削除

**処理：**
```javascript
MathEntryInteractionStateQuestion.prototype.removeAddGapBtn = function removeAddGapBtn() {
    $addGapBtn.off('click');
    $addGapBtn.remove();
};
```

#### 4.10.3 addMathFieldListener()

**目的：** MathField変更リスナーの追加

**処理：**
```javascript
MathEntryInteractionStateQuestion.prototype.addMathFieldListener = function addMathFieldListener() {
    var _widget = this.widget,
        interaction = _widget.element;

    interaction.onPci('responseChange', function (latex) {
        if (toBoolean(interaction.prop('useGapExpression'), false)) {
            interaction.prop('gapExpression', latex);
        } else {
            interaction.prop('gapExpression', '');
        }
    });
};
```

**イベント：** `responseChange`

**処理：**
- Gap Expressionモード → `gapExpression` プロパティを更新
- 通常モード → `gapExpression` をクリア

### 4.11 ヘルパー関数

#### 4.11.1 toBoolean(value, defaultValue)

**目的：** 値をBoolean型に変換

**実装：**
```javascript
function toBoolean(value, defaultValue) {
    if (typeof(value) === "undefined") {
        return defaultValue;
    } else {
        return (value === true || value === "true");
    }
}
```

#### 4.11.2 getToolsInitValues(interaction)

**目的：** ツールの初期値を取得

**戻り値：** `{ tool_frac: true, tool_sqrt: false, ... }`

**実装：**
```javascript
function getToolsInitValues(interaction) {
    return Object.keys(tools).reduce((acc, tool) => {
        acc[tool] = tools[tool].props.every(prop =>
            toBoolean(interaction.prop(prop.name), prop.defaultValue)
        );
        return acc;
    }, {});
}
```

#### 4.11.3 getToolsCallBacks()

**目的：** ツールの変更コールバックを生成

**戻り値：** `{ tool_frac: toolChangeCallBack, ... }`

**実装：**
```javascript
function getToolsCallBacks() {
    return Object.keys(tools).reduce((acc, tool) => {
        acc[tool] = toolChangeCallBack;
        return acc;
    }, {});
}
```

## 5. Correct State 仕様

### 5.1 概要

Map状態への遷移を制御する簡単な状態。

### 5.2 依存関係

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/factory',
    'taoQtiItem/qtiCreator/widgets/states/Correct'
], function (stateFactory, Correct) {
    // ...
});
```

### 5.3 InteractionStateCorrect

**ファクトリー構造：**
```javascript
var InteractionStateCorrect = stateFactory.create(
    Correct,
    function init() {
        this.widget.element.getResponseDeclaration().setTemplate('MAP_RESPONSE');
        this.widget.changeState('map');
    },
    function exit() {
        // 処理なし
    }
);
```

**処理：**
1. レスポンステンプレートを `MAP_RESPONSE` に設定
2. 即座に `map` 状態に遷移

## 6. Map State 仕様

### 6.1 概要

正解の定義、代替回答の管理、スコアリング設定を担当する最も複雑な状態。

### 6.2 依存関係

```javascript
define([
    'handlebars',
    'i18n',
    'lodash',
    'jquery',
    'taoQtiItem/qtiCreator/widgets/states/factory',
    'taoQtiItem/qtiCreator/widgets/states/Map',
    'tpl!mathEntryInteraction/creator/tpl/responseForm',
    'tpl!mathEntryInteraction/creator/tpl/scoreForm',
    'tpl!mathEntryInteraction/creator/tpl/addAlternativeBtn',
    'tpl!mathEntryInteraction/creator/tpl/alternativeForm',
    'taoQtiItem/qtiCreator/widgets/component/minMax/minMax',
    'taoQtiItem/qtiCreator/widgets/helpers/formElement',
    'ui/tooltip',
    'mathEntryInteraction/runtime/helper/ambiguousSymbols'
], function(...) {
    // ...
});
```

### 6.3 グローバル変数

```javascript
let uidCounter = 0;
let defaultScoreValue = 1;
```

### 6.4 Handlebars ヘルパー

```javascript
hb.registerHelper('increaseIndex', function (value) {
    return parseInt(value) + 1;
});
```

### 6.5 State Factory

```javascript
const MathEntryInteractionStateResponse = stateFactory.create(
    MapState,
    function init() {
        this.initGlobalVariables();
        this.initForm();
    },
    function exit() {
        this.emptyGapFields();
        this.toggleResponseMode(false);
        this.saveAnswers();
        this.removeResponseChangeEventListener();
        this.removeDeleteListeners();
        this.removeAddButtonListener();
        this.destroyForm();
    }
);
```

### 6.6 プロパティ

| プロパティ | 型 | 説明 |
|-----------|---|------|
| `activeEditId` | String | 現在編集中の回答ID |
| `correctResponses` | Map | 正解レスポンスマネージャ |
| `gapTemplate` | String | Gap Expressionテンプレート |

### 6.7 初期化メソッド

#### 6.7.1 initGlobalVariables()

**目的：** グローバル変数の初期化

**処理：**
```javascript
initGlobalVariables: function initGlobalVariables() {
    let interaction = this.widget.element;
    this.activeEditId = null;

    // PCIのresponses managerを取得
    const pci = this.widget.element.data('pci');
    const responsesManager = pci.getResponsesManager();

    this.correctResponses = responsesManager;

    // UID カウンターをリセット
    const [inputIndex] = this.correctResponses.keys();
    const inputValue = this.correctResponses.get(inputIndex);

    if (inputIndex.length) {
        uidCounter = inputIndex.split('-')[1];
        uidCounter++;
    }

    // 既存の代替回答をクリア
    this.correctResponses.clear();
    $('.math-entry-alternative-wrap').parent('div').remove();

    this.correctResponses.set(inputIndex, inputValue);

    // Gap Expressionテンプレートを保存
    if (this.inGapMode() === true) {
        interaction = this.widget.element;
        this.gapTemplate = interaction.prop('gapExpression');
    }
}
```

#### 6.7.2 initForm()

**目的：** レスポンスフォームの初期化

**処理フロー：**
```
1. responseForm テンプレートをレンダリング
2. min/max コンポーネントを初期化
3. スコアレスポンスUIを作成
4. レスポンスフォームを初期化
5. 編集オプションを初期化
6. 代替入力を初期化
7. ツールチップを表示
```

**実装：**
```javascript
initForm: function initForm() {
    const interaction = this.widget.element;
    const $responseForm = this.widget.$responseForm;
    const response = interaction.getResponseDeclaration();
    const mappingDisabled = false;

    this.initResponseChangeEventListener();

    // responseForm レンダリング
    $responseForm.html(responseFormTpl({
        identifier: interaction.attr('responseIdentifier'),
        serial: response.serial,
        min: interaction.prop('min'),
        max: interaction.prop('max'),
        mappingDisabled,
        defaultValue: response.getMappingAttribute('defaultValue')
    }));

    // min/max コンポーネント
    minMaxComponentFactory($responseForm.find('.response-mapping-attributes > .min-max-panel'), {
        min: {
            fieldName: 'lowerBound',
            value: _.parseInt(response.getMappingAttribute('lowerBound')) || 0,
            helpMessage: __('Minimal score for this interaction.')
        },
        max: {
            fieldName: 'upperBound',
            value: _.parseInt(response.getMappingAttribute('upperBound')) || 0,
            helpMessage: __('Maximal score for this interaction.')
        },
        upperThreshold: Number.MAX_SAFE_INTEGER,
        syncValues: true
    });

    this.createScoreResponse();
    this.initResponseForm();
    this.initEditingOptions();
    this.initAlternativeInput();

    tooltip.lookup($responseForm);
}
```

#### 6.7.3 initResponseForm()

**目的：** 既存の正解データをロードして初期化

**処理：**
```javascript
initResponseForm: function initResponseForm() {
    const interaction = this.widget.element;
    let newCorrectAnswer;

    // 最初のinput IDを取得
    const $input = this.widget.$container.find('.math-entry-input');
    const $score = this.widget.$container.find('.math-entry-response-wrap .math-entry-score-input');

    let id = this.correctResponses.getIndex();
    $score[0].dataset.for = id;
    $input[0].dataset.index = id;

    // Gap Expressionモードの場合、空のgap値を生成
    if (this.inGapMode() === true) {
        this.emptyGapFields();
        const gapExpression = interaction.prop('gapExpression');
        const gapCount = (gapExpression.match(/\\taoGap/g) || []).length;

        if (gapCount > 0) {
            newCorrectAnswer = [];
            for (let i = 0; i < gapCount; i++) {
                newCorrectAnswer.push(' ');
            }
            newCorrectAnswer = newCorrectAnswer.join(',');
        } else {
            newCorrectAnswer = '';
        }
    } else {
        newCorrectAnswer = '';
        this.toggleResponseMode(false);
    }

    // 既存の正解を取得
    let existingReponses = this.getExistingCorrectAnswerOptions();
    const response = interaction.getResponseDeclaration();

    if (existingReponses && existingReponses.length) {
        // 既存の正解を処理
        const correctResponse = response.getCorrect();
        if (correctResponse && correctResponse.length) {
            let correctResponseIndex = existingReponses.indexOf(correctResponse[0]);
            if (correctResponseIndex !== -1) {
                existingReponses.splice(correctResponseIndex, 1);
                existingReponses.unshift(correctResponse[0]);
            }
        }

        const mapEntries = response.getMapEntries();
        existingReponses.forEach((entry, index) => {
            let newId = id;
            if (index > 0) {
                newId = this.uid();
            }
            const inputValue = this.checkValues(newId);
            this.correctResponses.set(newId, Object.assign(inputValue, { response: entry }));

            if (mapEntries[entry] && index < 1) {
                $score[0].value = mapEntries[entry];
            }
        });
    } else {
        // 新規の場合
        const inputValue = this.checkValues(id);
        this.correctResponses.set(id, Object.assign(inputValue, { response: newCorrectAnswer }));
    }

    this.activeEditId = id;
}
```

### 6.8 スコアリングメソッド

#### 6.8.1 createScoreResponse()

**目的：** スコア入力UIの作成

**処理：**
```javascript
createScoreResponse: function createScoreResponse() {
    const $addAlternativeBtn = $(addAlternativeBtn());
    const $container = this.widget.$container;
    const interaction = this.widget.element;
    const response = interaction.getResponseDeclaration();

    // スコア入力フィールドのイベント
    const $score = $container.find('.math-entry-response-wrap .math-entry-score-input');
    $score.on('click', e => {
        e.stopPropagation();
        e.preventDefault();
    });

    // defaultValue変更時のプレースホルダー更新
    this.widget.on('mappingAttributeChange', data => {
        if (data.key === 'defaultValue') {
            $score.attr('placeholder', data.value);
        }
    });

    // フォーム初期化
    formElement.initWidget($container);
    formElement.setChangeCallbacks($container, response, {
        mathEntryScoreInput: (rsp, value) => {
            const key = $(this.widget).data('for');
            if (value === '') {
                rsp.removeMapEntry(key);
            } else {
                rsp.setMapEntry(key, value, true);
            }
        }
    });

    // スコアフォームテンプレートを挿入
    if ($container.find('.math-entry-response-wrap').length > 0) {
        return false;
    }

    const $input = $container.find('.math-entry-input');
    const parent = $input[0].parentNode;
    $(parent).prepend(scoreTpl({
        placeholder: response.getMappingAttribute('defaultValue'),
        score: this.getScoreValue()
    }));

    const $correct = $container.find('.math-entry-correct-wrap');
    $input.detach().appendTo($correct);
    $(parent).append($addAlternativeBtn);
}
```

#### 6.8.2 getScoreValue()

**目的：** デフォルトスコア値を取得

**実装：**
```javascript
getScoreValue: function getScoreValue() {
    const response = this.widget.element.getResponseDeclaration();
    const getMappingDefault = response.getMappingAttribute('defaultValue');

    if (+getMappingDefault !== 0) {
        defaultScoreValue = getMappingDefault;
    } else {
        defaultScoreValue = 1;
    }

    return defaultScoreValue;
}
```

### 6.9 代替回答管理

#### 6.9.1 initAlternativeInput()

**目的：** "Add Alternative" ボタンのイベントを登録

**実装：**
```javascript
initAlternativeInput: function initAlternativeInput() {
    this.widget.$container.find('.math-entry-response-correct.btn-info').on('click', () => {
        this.addAlternativeInput(null);
    });
}
```

#### 6.9.2 addAlternativeInput(responseId)

**目的：** 代替回答入力フィールドの追加

**パラメータ：**
- `responseId` (String, optional): レスポンスID（既存の場合）

**処理フロー：**
```
1. UID生成（既存または新規）
2. Gap値またはLaTeX値を設定
3. alternativeForm テンプレートを挿入
4. responsesManager に追加
5. PCI イベントを発火 ('addAlternative')
6. 削除オプションを初期化
```

**実装（一部）：**
```javascript
addAlternativeInput: function addAlternativeInput(responseId) {
    const interaction = this.widget.element;
    const $container = this.widget.$container;
    const response = interaction.getResponseDeclaration();
    const mapEntries = response.getMapEntries();
    let responseValue = '';
    const id = responseId || this.uid();

    let gapValues = '';
    let scoreValue = this.getScoreValue();

    // Gap Expressionモードの処理
    if (this.inGapMode() === true) {
        if (this.correctResponses.has(responseId)) {
            gapValues = {
                base: {
                    string: this.correctResponses.get(responseId).response
                }
            };
        } else {
            const gapExpression = this.gapTemplate;
            const gapCount = (gapExpression.match(/\\taoGap/g) || []).length;

            if (gapCount > 0) {
                gapValues = [];
                for (let i = 0; i < gapCount; i++) {
                    gapValues.push(' ');
                }
                gapValues = {
                    base: {
                        string: gapValues.join(',')
                    }
                };
            } else {
                gapValues = {
                    base: {
                        string: gapValues
                    }
                };
            }
        }

        responseValue = this.gapTemplate;
        this.correctResponses.set(id, { response: gapValues });

        if (!!responseId && Object.keys(mapEntries).length > 0) {
            scoreValue = mapEntries[this.correctResponses.get(responseId).response.base.string];
        }
    }
    // 通常モードの処理
    else {
        let value = this.correctResponses.has(responseId) &&
                    this.correctResponses.get(responseId).response;
        responseValue = value || '';
        this.correctResponses.set(id, { response: responseValue });

        if (!!responseId && Object.keys(mapEntries).length > 0) {
            scoreValue = mapEntries[this.correctResponses.get(responseId).response];
        }
    }

    // 代替番号の計算
    let alternativeNumber = 1;
    const alternativeInputs = $container.find('.math-entry-alternative-input');

    if (!!responseId && alternativeInputs.length) {
        const alternativeInput = this.correctResponses.get(responseId).input;
        if (!!alternativeInput) {
            $.each(alternativeInputs, (index, input) => {
                if (input === alternativeInput) {
                    alternativeNumber = index;
                }
            });
        } else {
            alternativeNumber = alternativeInputs.length + 1;
        }
    } else if (alternativeInputs.length) {
        alternativeNumber = alternativeInputs.length + 1;
    }

    // テンプレート挿入
    $container.find('button.math-entry-response-correct', $container).before(alternativeFormTpl({
        index: id,
        placeholder: response.getMappingAttribute('defaultValue'),
        score: scoreValue,
        alternativeNumber: alternativeNumber
    }));

    // フォーム初期化（スコア入力）
    // ...

    // PCI イベント発火
    interaction.triggerPci('addAlternative', [responseValue, gapValues, id]);
    this.initDeletingOptions();
}
```

### 6.10 レスポンス変更イベント

#### 6.10.1 initResponseChangeEventListener()

**目的：** レスポンス変更イベントのリスナーを登録

**実装：**
```javascript
initResponseChangeEventListener: function initResponseChangeEventListener() {
    const interaction = this.widget.element;

    interaction.onPci('responseChange', (latex, index) => {
        if (interaction.prop('inResponseState')) {
            let editIdIndex = null;

            if (index && index.length > 0) {
                editIdIndex = index;
            } else if (!!this.activeEditId) {
                editIdIndex = this.activeEditId;
            }

            this.correctResponses.currentIndex(editIdIndex);

            // 通常モード
            if (this.inGapMode(this) === false && editIdIndex !== null) {
                const inputValue = this.checkValues(editIdIndex);
                this.correctResponses.set(editIdIndex,
                    Object.assign(inputValue, { response: latex }));
            }
            // Gap Expressionモード
            else if (this.inGapMode(this) === true && editIdIndex !== null) {
                const response = interaction.getResponse(editIdIndex);
                if (response !== null) {
                    if (response.base.string.length > 0 || !!editIdIndex) {
                        const inputValue = this.checkValues(editIdIndex);
                        this.correctResponses.set(editIdIndex,
                            Object.assign(inputValue, { response: response.base.string }));
                    }
                }
            }
        }
    });
}
```

#### 6.10.2 removeResponseChangeEventListener()

**目的：** イベントリスナーの削除

**実装：**
```javascript
removeResponseChangeEventListener: function removeResponseChangeEventListener() {
    const interaction = this.widget.element;
    interaction.offPci('responseChange');
}
```

### 6.11 正解の保存

#### 6.11.1 saveAnswers()

**目的：** 全ての正解をResponseDeclarationに保存

**処理フロー：**
```
1. 既存のMapEntriesをクリア
2. 全正解回答を取得
3. スコア値を取得
4. 各回答について:
   - 曖昧な記号を正規化
   - setMapEntry(answer, score)
5. 最初の回答をsetCorrect()で設定
```

**実装：**
```javascript
saveAnswers: function saveAnswers() {
    const interaction = this.widget.element;
    const responseDeclaration = interaction.getResponseDeclaration();

    this.clearMapEntries();

    let gapResponses = new Map();
    if (this.inGapMode() === true) {
        this.correctResponses.forEach((value, index) => {
            let response = ' ';
            if (value.response && value.response.length &&
                value.response.split(',').indexOf('') === -1) {
                response = value.response;
            }
            gapResponses.set(index, response);
        });
    }

    const score = new Map();
    if (this.correctResponses.size) {
        const scoreInput = this.widget.$container.find('.math-entry-score-input');
        $(scoreInput).each(input => {
            score.set(scoreInput[input].dataset.for, scoreInput[input].value);
        });
    }

    this.correctResponses.forEach((response, index) => {
        const scoreValue = score.get(index) ||
                          responseDeclaration.getMappingAttribute('defaultValue');

        responseDeclaration.setMapEntry(
            convertAmbiguousSymbols(response.response) || ' ',
            scoreValue,
            false
        );

        const [correctIndex] = this.correctResponses.keys();
        responseDeclaration.setCorrect(
            convertAmbiguousSymbols(this.correctResponses.get(correctIndex).response) || ' '
        );
    });
}
```

#### 6.11.2 clearMapEntries()

**目的：** 既存のMapEntriesをクリア

**実装：**
```javascript
clearMapEntries: function clearMapEntries() {
    const interaction = this.widget.element;
    const response = interaction.getResponseDeclaration();
    const mapEntries = response.getMapEntries();

    _.keys(mapEntries).forEach(function (mapKey) {
        response.removeMapEntry(mapKey, true);
    });
}
```

### 6.12 編集オプション

#### 6.12.1 initEditingOptions()

**目的：** 既存の正解回答を編集可能な状態にする

**処理：**
```javascript
initEditingOptions: function initEditingOptions() {
    this.toggleResponseMode(true);

    const interaction = this.widget.element;
    const responseDeclaration = interaction.getResponseDeclaration();
    const mapEntries = responseDeclaration.getMapEntries();
    const [correctIndex] = this.correctResponses.keys();

    if (this.correctResponses.size > 0) {
        this.correctResponses.forEach((value, index) => {
            this.activeEditId = index;
            this.correctResponses.currentIndex(index);

            // Gap Expressionモード
            if (this.inGapMode() === true) {
                if (!Object.keys(value).includes('input') && index !== correctIndex) {
                    this.addAlternativeInput(index);
                } else {
                    const response = this.getGapResponseObject(value.response || '');
                    interaction.triggerPci('latexGapInput', [response, index]);
                }
            }
            // 通常モード
            else {
                if (!Object.keys(value).includes('input') && index !== correctIndex) {
                    this.addAlternativeInput(index);
                } else {
                    interaction.triggerPci('latexInput', [value.response || '', index]);

                    const scoreInput = this.widget.$container
                        .find('.math-entry-score-input.math-entry-response-correct');

                    if (scoreInput.length > 0 && !!index && value.response) {
                        scoreInput[0].value = mapEntries && mapEntries[value.response];
                    }
                }
            }
        });
    }

    this.activeEditId = null;
}
```

#### 6.12.2 getGapResponseObject(response)

**目的：** Gap形式のレスポンスオブジェクトを生成

**パラメータ：**
- `response` (String): カンマ区切りのLaTeX文字列

**戻り値：**
```javascript
{
    base: {
        string: ['gap1', 'gap2', 'gap3']
    }
}
```

**実装：**
```javascript
getGapResponseObject: function getGapResponseObject(response) {
    return {
        base: {
            string: response.split(',')
        }
    };
}
```

### 6.13 削除機能

#### 6.13.1 initDeletingOptions()

**目的：** 削除イベントのリスナーを登録

**実装：**
```javascript
initDeletingOptions: function initDeletingOptions() {
    const interaction = this.widget.element;

    interaction.onPci('deleteInput', (inputId) => {
        if (this.inGapMode() === true) {
            this.emptyGapFields();
        }
        this.activeEditId = null;
    });
}
```

#### 6.13.2 removeDeleteListeners()

**目的：** 削除リスナーのクリーンアップ

**実装：**
```javascript
removeDeleteListeners: function removeDeleteListeners() {
    const $deleteButtons = this.widget.$container.find('.entry-config');
    $deleteButtons.find('.answer-delete').off('click');
}
```

### 6.14 クリーンアップメソッド

#### 6.14.1 destroyForm()

**実装：**
```javascript
destroyForm: function destroyForm() {
    const $responseForm = this.widget.$responseForm;
    $responseForm.find('.mathEntryInteraction').remove();
}
```

#### 6.14.2 removeAddButtonListener()

**実装：**
```javascript
removeAddButtonListener: function removeAddButtonListener() {
    this.widget.$container.find('.math-entry-response-correct.btn-info').off('click');
}
```

### 6.15 ヘルパーメソッド

#### 6.15.1 uid()

**目的：** 一意なIDを生成

**実装：**
```javascript
uid: function uid() {
    return `answer-${uidCounter++}`;
}
```

#### 6.15.2 checkValues(index)

**目的：** 既存の入力値をチェック

**実装：**
```javascript
checkValues: function (index) {
    let inputValue = {};
    if (this.correctResponses.has(index) &&
        Object.keys(this.correctResponses.get(index)).includes('input')) {
        inputValue = { input: this.correctResponses.get(index).input };
    }
    return inputValue;
}
```

#### 6.15.3 inGapMode()

**実装：**
```javascript
inGapMode: function inGapMode() {
    const interaction = this.widget.element;
    const useGapExpression = interaction.prop('useGapExpression');
    return useGapExpression && useGapExpression !== 'false' || false;
}
```

#### 6.15.4 toggleResponseMode(value)

**実装：**
```javascript
toggleResponseMode: function toggleResponseMode(value) {
    const interaction = this.widget.element;

    if (interaction.prop('inResponseState') !== value) {
        interaction.prop('inResponseState', value);
        interaction.triggerPci('configChange', [interaction.getProperties()]);
    }
}
```

#### 6.15.5 emptyGapFields()

**実装：**
```javascript
emptyGapFields: function emptyGapFields() {
    const interaction = this.widget.element;

    if (this.inGapMode() === true) {
        this.activeEditId = null;
        interaction.prop('gapExpression', this.gapTemplate);
    }
}
```

#### 6.15.6 getExistingCorrectAnswerOptions()

**目的：** 既存の正解オプションを取得

**実装：**
```javascript
getExistingCorrectAnswerOptions: function getExistingCorrectAnswerOptions() {
    const interaction = this.widget.element;
    const mapEntries = interaction.getResponseDeclaration().getMapEntries();
    return _.keys(mapEntries) || [];
}
```

## 変更履歴

| バージョン | 日付 | 変更内容 |
|-----------|------|---------|
| 2.8.0 | 2026-01-11 | 初版作成 |
