# Math Entry Interaction - Creator 状態管理の詳細解説

## 概要

TAO QTI Item Creator は **State Pattern（状態パターン）** を使用してアイテム編集UIを管理しています。Math Entry Interaction では以下の3つの状態があります。

---

## 状態遷移図

```
┌─────────────────────────────────────────────────────────┐
│                     Widget.js                            │
│  （ベースクラス - 状態マシンの管理者）                    │
│  - 3つの状態を登録                                       │
│  - 状態遷移を制御                                        │
│  - インタラクションの初期化                               │
└─────────────────────────────────────────────────────────┘
                           │
                           │ 登録
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    states.js                             │
│  （状態バンドル - 3つの状態をまとめる）                  │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Question   │  │   Correct   │  │     Map     │    │
│  │   状態      │  │    状態     │  │    状態     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
       │                    │                    │
       │                    │                    │
       ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ Question.js │      │ Correct.js  │      │   Map.js    │
│ (500行)     │      │  (35行)     │      │  (700行)    │
└─────────────┘      └─────────────┘      └─────────────┘
```

---

## ユーザーの操作と状態遷移

### TAO Item Editor の UI

```
┌──────────────────────────────────────────────────────────┐
│  TAO Item Editor                                         │
├──────────────────────────────────────────────────────────┤
│  [Question] [Correct Response] [Response Processing]     │  ← タブ
│  ─────────  ──────────────────  ────────────────────     │
│                                                           │
│  ┌────────────────────────────────────────────────┐     │
│  │  ここに各状態のUIが表示される                   │     │
│  │                                                 │     │
│  │  Question状態: プロパティフォーム               │     │
│  │  Correct状態: （即座にMapに遷移）               │     │
│  │  Map状態: 正答・別解・スコア設定フォーム        │     │
│  └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### 状態遷移の流れ

```
ユーザーが "Question" タブをクリック
          ↓
    Question状態に遷移
          ↓
    create() 関数実行（初期化）
          ↓
    - プロパティフォーム表示
    - プロンプトエディタ有効化
    - Gap追加ボタン表示（useGapExpression=trueの場合）
          ↓
    ユーザーがプロパティを編集
          ↓
    他のタブをクリック
          ↓
    exit() 関数実行（クリーンアップ）

───────────────────────────────────────────────────────

ユーザーが "Correct Response" タブをクリック
          ↓
    Correct状態に遷移
          ↓
    init() 関数実行
          ↓
    即座にMap状態に遷移（this.widget.changeState('map')）

───────────────────────────────────────────────────────

Map状態に遷移（自動 or "Response Processing" タブクリック）
          ↓
    init() 関数実行（初期化）
          ↓
    - グローバル変数初期化
    - レスポンスフォーム表示
    - 正答入力MathField作成
    - 別解追加ボタン表示
    - イベントリスナー登録
          ↓
    ユーザーが正答・別解・スコアを設定
          ↓
    他のタブをクリック
          ↓
    exit() 関数実行（クリーンアップ）
          ↓
    - 入力内容を保存
    - MathField破棄
    - イベントリスナー削除
```

---

## 各ファイルの詳細

### 1. Widget.js（ベースクラス）

**ファイルパス**: `creator/widget/Widget.js`

**行数**: 41行

**役割**:
- **状態マシンの管理者**
- 3つの状態（Question, Correct, Map）を登録
- インタラクションの初期化

**主要コード**:
```javascript
var MathEntryInteractionWidget = Widget.clone();

MathEntryInteractionWidget.initCreator = function initCreator() {
    // 1. 状態を登録
    this.registerStates(states);  // states.jsから3つの状態をインポート

    // 2. 親クラスの初期化
    Widget.initCreator.call(this);

    // 3. CSS クラス追加
    $interaction = this.$container.find('.mathEntryInteraction');
    if ($interaction.length) {
        $interaction.addClass('tao-qti-creator-context');
    }
};
```

**いつ実行されるか**:
- アイテム作成者がMath Entry Interactionをアイテムに追加したとき
- アイテムを開いてMath Entry Interactionを選択したとき

**主な機能**:
1. **状態登録**: `registerStates(states)` で3つの状態を登録
2. **初期化**: 親クラスのinitCreatorを呼び出し
3. **CSS設定**: Creatorコンテキストを示すCSSクラスを追加

---

### 2. states.js（状態バンドル）

**ファイルパス**: `creator/widget/states/states.js`

**行数**: 29行

**役割**:
- **3つの状態をバンドルして提供**
- TAOの状態ファクトリーを使用して状態セットを作成

**主要コード**:
```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/factory',
    'taoQtiItem/qtiCreator/widgets/interactions/customInteraction/states/states',
    'mathEntryInteraction/creator/widget/states/Question',
    'mathEntryInteraction/creator/widget/states/Correct',
    'mathEntryInteraction/creator/widget/states/Map'
], function(factory, states){
    'use strict';

    // 3つの状態をバンドル
    return factory.createBundle(states, arguments);
});
```

**3つの状態**:
1. **Question**: プロパティ設定
2. **Correct**: 正答設定への中継（即座にMapに遷移）
3. **Map**: 実際の正答・採点設定

**いつ実行されるか**:
- Widget.jsがregisterStates()を呼び出したとき

---

### 3. Question.js（Question状態）★重要

**ファイルパス**: `creator/widget/states/Question.js`

**行数**: 約500行

**役割**:
- **問題のプロパティを設定する状態**
- ツールボタン（80+個）の有効/無効設定
- Gap Expressionモードの切り替え
- プロンプトの編集

**いつ実行されるか**:
- ユーザーが **"Question"** タブをクリックしたとき
- インタラクション作成直後（デフォルト状態）

**ライフサイクル**:

#### create() 関数（初期化）

```javascript
var MathEntryInteractionStateQuestion = stateFactory.extend(Question, function create(){
    var $container = this.widget.$container,
        $prompt = $container.find('.prompt'),
        interaction = this.widget.element;

    // 1. プロンプトエディタを有効化
    containerEditor.create($prompt, {
        change: function(text){
            interaction.data('prompt', text);
            interaction.updateMarkup();
            mathInPrompt.postRender($prompt);  // MathML表示
        },
        // ...
    });

    // 2. Gap Expressionモードならボタン追加
    if (toBoolean(interaction.prop('useGapExpression'), false)) {
        this.createAddGapBtn();
    }

    // 3. MathFieldリスナー追加
    this.addMathFieldListener();
},
```

**実行内容**:
1. **プロンプトエディタ有効化**: ユーザーが問題文を編集できるようにする
2. **Gap追加ボタン表示**: Gap Expressionモードの場合
3. **MathFieldリスナー登録**: 数式フィールドの変更を監視

#### exit() 関数（クリーンアップ）

```javascript
function exit(){
    var $container = this.widget.$container,
        $prompt = $container.find('.prompt');

    // 1. エディタ破棄
    simpleEditor.destroy($container);
    containerEditor.destroy($prompt);

    // 2. Gap追加ボタン削除
    this.removeAddGapBtn();
});
```

**実行内容**:
1. **エディタ破棄**: メモリリーク防止
2. **ボタン削除**: UIクリーンアップ

**主要メソッド**:

```javascript
// プロパティフォーム表示
MathEntryInteractionStateQuestion.prototype.initForm = function() {
    // 1. フォームHTML生成（Handlebars）
    $form.html(formTpl(_.assign({
        serial: response.serial,
        identifier: interaction.attr('responseIdentifier'),
        authorizeWhiteSpace: toBoolean(...),
        useGapExpression: toBoolean(...),
        // ... 80+ のツールボタン設定
    }, getToolsInitValues(interaction))));

    // 2. フォーム要素初期化（バリデーション等）
    formElement.initWidget($form);

    // 3. 変更コールバック設定
    formElement.setChangeCallbacks($form, interaction, {
        identifier: function(i, value) { /* ... */ },
        useGapExpression: function(i, value) { /* ... */ },
        tool_frac: function(i, value) { /* ... */ },
        // ... 80+ のコールバック
    });
};
```

**80+ツールボタン管理**:

```javascript
const tools = {
    tool_frac:       { props: [{ name: 'tool_frac', defaultValue: true }]},
    tool_sqrt:       { props: [{ name: 'tool_sqrt', defaultValue: true }]},
    tool_exp:        { props: [{ name: 'tool_exp', defaultValue: true }]},
    // ... 80+ のツール定義
};
```

**Gap Expression切り替え**:

```javascript
useGapExpression: function gapChangeCallback(i, value) {
    if (toBoolean(value, false)) {
        self.createAddGapBtn();   // Gap追加ボタン作成
        $gapStyleBox.show();      // Gapサイズ選択表示
        response.removeMapEntries();  // 既存マッピング削除
    } else {
        self.removeAddGapBtn();   // Gap追加ボタン削除
        $gapStyleBox.hide();      // Gapサイズ選択非表示
        response.removeMapEntries();
    }
}
```

**表示されるUI**:
- Response Identifier（レスポンスID入力）
- Options（オプション設定）
  - authorize white space（空白許可）
  - use expression with gaps（Gap Expression有効化）
  - Gap size（Gap サイズ: Small/Medium/Large）
- Tools（ツールボタン選択）
  - Enable all symbols（全ツール一括ON/OFF）
  - 各ツールグループ（Basic, Functions, Greek, Geometry, etc.）

---

### 4. Correct.js（Correct状態）

**ファイルパス**: `creator/widget/states/Correct.js`

**行数**: 35行

**役割**:
- **Map状態への中継役**
- Response Processingテンプレートを設定
- 即座にMap状態に遷移

**いつ実行されるか**:
- ユーザーが **"Correct Response"** タブをクリックしたとき

**主要コード**:
```javascript
var InteractionStateCorrect = stateFactory.create(
    Correct,
    function init() {
        // 1. Response Processingテンプレートを設定
        this.widget.element.getResponseDeclaration().setTemplate('MAP_RESPONSE');

        // 2. 即座にMap状態に遷移
        this.widget.changeState('map');
    },
    function exit() {
        // 何もしない（Map状態が処理を行う）
    }
);
```

**なぜこの状態が存在するか**:
1. **Response Processingテンプレート設定**: TAOの採点エンジンに「MAP_RESPONSE」テンプレートを使用することを伝える
2. **統一されたインターフェース**: TAOの標準的なインタラクションとの互換性維持
3. **将来の拡張性**: 必要に応じてCorrect状態固有の処理を追加可能

**実際の動作**:
```
ユーザーが "Correct Response" タブクリック
    ↓
Correct状態のinit()実行
    ↓
setTemplate('MAP_RESPONSE')  ← 採点方式を設定
    ↓
changeState('map')  ← 即座にMap状態に遷移
    ↓
Map状態のinit()実行
```

---

### 5. Map.js（Map状態）★最重要

**ファイルパス**: `creator/widget/states/Map.js`

**行数**: 約700行

**役割**:
- **正答・別解・スコアを設定する状態**
- Response Processing（採点処理）の設定
- 最も複雑な状態

**いつ実行されるか**:
- Correct状態から自動遷移したとき
- ユーザーが **"Response Processing"** タブをクリックしたとき

**ライフサイクル**:

#### init() 関数（初期化）

```javascript
const MathEntryInteractionStateResponse = stateFactory.create(
    MapState,
    function init() {
        this.initGlobalVariables();  // グローバル変数初期化
        this.initForm();             // フォーム初期化
    },
```

**実行内容**:

**1. グローバル変数初期化**:
```javascript
initGlobalVariables: function() {
    let interaction = this.widget.element;
    this.activeEditId = null;

    // PCI からレスポンスマネージャー取得
    const pci = this.widget.element.data('pci');
    const responsesManager = pci.getResponsesManager();

    this.correctResponses = responsesManager;  // ES6 Map

    // Gap Expressionモードの場合
    if (this.inGapMode() === true) {
        this.gapTemplate = interaction.prop('gapExpression');
    }
}
```

**2. フォーム初期化**:
```javascript
initForm: function() {
    const interaction = this.widget.element;
    const $responseForm = this.widget.$responseForm;

    // 1. レスポンスフォームHTML生成
    $responseForm.html(responseFormTpl({
        identifier: interaction.attr('responseIdentifier'),
        serial: response.serial,
        min: interaction.prop('min'),
        max: interaction.prop('max'),
        defaultValue: response.getMappingAttribute('defaultValue')
    }));

    // 2. 正答フォーム追加
    this.addCorrectAnswer();

    // 3. 別解追加ボタン表示
    this.addAlternativeButton();

    // 4. イベントリスナー登録
    this.initResponseChangeEventListener();
    this.initDeleteListeners();
    this.initAddButtonListener();
}
```

#### exit() 関数（クリーンアップ）

```javascript
function exit() {
    this.emptyGapFields();                      // Gapフィールド空にする
    this.toggleResponseMode(false);              // レスポンスモード無効化
    this.saveAnswers();                          // 正答・別解を保存
    this.removeResponseChangeEventListener();    // イベントリスナー削除
    this.removeDeleteListeners();
    this.removeAddButtonListener();
    this.destroyForm();                          // フォーム破棄
}
```

**主要メソッド**:

**正答追加**:
```javascript
addCorrectAnswer: function() {
    const index = this.uid();  // 'answer-0', 'answer-1', ...
    const scoreValue = this.getScoreValue();

    // 1. 正答フォームHTML追加
    const $answerWrap = $(answerFormTpl({
        uid: index,
        label: __('Correct'),
        deletable: false
    }));

    // 2. スコアフォーム追加
    const $scoreForm = $(scoreTpl({
        score: scoreValue,
        placeholder: scoreValue
    }));

    // 3. MathFieldプレースホルダー作成
    const $mathFieldPlaceholder = $answerWrap.find('.math-entry-input');

    // 4. ResponsesManagerに登録
    this.correctResponses.set(index, {
        input: $mathFieldPlaceholder,
        score: scoreValue,
        deletable: false
    });
}
```

**別解追加**:
```javascript
addAlternative: function() {
    const index = this.uid();
    const scoreValue = this.getScoreValue();

    // 1. 別解フォームHTML追加
    const $alternativeWrap = $(alternativeFormTpl({
        uid: index,
        label: __('Alternative'),
        deletable: true  // 削除可能
    }));

    // 2. スコアフォーム追加
    // 3. MathFieldプレースホルダー作成
    // 4. ResponsesManagerに登録

    // 5. 削除ボタンリスナー追加
    this.initDeleteListeners();
}
```

**別解削除**:
```javascript
deleteAnswer: function(index) {
    // 1. ResponsesManagerから削除
    this.correctResponses.delete(index);

    // 2. DOM要素削除
    $(`.math-entry-alternative-wrap[data-uid="${index}"]`).remove();

    // 3. Response Declarationから削除
    const response = this.widget.element.getResponseDeclaration();
    response.removeMapEntry(convertedLatex);
}
```

**正答・別解の保存**:
```javascript
saveAnswers: function() {
    const response = this.widget.element.getResponseDeclaration();

    // 既存のマッピングをクリア
    response.removeMapEntries();

    // 各正答・別解をループ
    this.correctResponses.forEach((value, index) => {
        const latex = this.getLatexFromMathField(value.input);
        const convertedLatex = convertAmbiguousSymbols(latex);
        const score = value.score;

        // Response Declarationに追加
        response.setMapEntry(convertedLatex, score, true);
    });
}
```

**表示されるUI**:
- **Response Form**:
  - Response identifier
  - Scoring mode
  - Mapping attributes（Lower bound, Upper bound, Default value）

- **Correct Answer**:
  - 正答入力用MathField
  - Score入力フィールド

- **Alternatives**（別解）:
  - 別解入力用MathField（複数）
  - 各別解のScore
  - 削除ボタン
  - "Add alternative" ボタン

---

## データフロー

### プロパティ設定のフロー（Question状態）

```
ユーザーがツールボタンチェックボックスをクリック
    ↓
formElement.setChangeCallbacks で登録されたコールバック実行
    ↓
configChangeCallBack(interaction, value, name)
    ↓
interaction.prop(name, value)  // プロパティ保存
    ↓
interaction.triggerPci('configChange', [properties])
    ↓
Runtime側でconfigChangeイベントを受信
    ↓
ツールバーを再生成
```

### 正答設定のフロー（Map状態）

```
ユーザーが正答MathFieldに数式を入力
    ↓
MathField の 'edit' イベント発火
    ↓
this.correctResponses.set(index, {
    input: $mathField,
    score: scoreValue,
    deletable: false
})
    ↓
ユーザーが他のタブをクリック（状態遷移）
    ↓
exit() 関数実行
    ↓
saveAnswers() 実行
    ↓
各MathFieldからLaTeX取得
    ↓
convertAmbiguousSymbols() で記号正規化
    ↓
response.setMapEntry(latex, score, true)
    ↓
Response Declarationに保存
```

### 別解追加のフロー（Map状態）

```
ユーザーが "Add alternative" ボタンをクリック
    ↓
addAlternativeButtonListener 実行
    ↓
addAlternative() 関数実行
    ↓
UID生成（'answer-1', 'answer-2', ...）
    ↓
別解フォームHTML生成・追加
    ↓
MathFieldプレースホルダー作成
    ↓
this.correctResponses.set(uid, {...})
    ↓
削除ボタンリスナー追加
```

---

## 状態間の関係性

### 継承関係

```
TAO Core の基底状態
    ↓
mathEntryInteraction固有の状態

具体的には:
- Question.js は taoQtiItem/.../states/Question を継承
- Correct.js は taoQtiItem/.../states/Correct を継承
- Map.js は taoQtiItem/.../states/Map を継承
```

### 状態遷移

```
┌──────────────────────────────────────────────────────┐
│                   初期状態                            │
│              （インタラクション追加時）                │
└──────────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────┐
│                  Question状態                         │
│  - プロパティフォーム表示                              │
│  - プロンプト編集可能                                  │
│  - ツールボタン選択                                    │
│  - Gap Expression切り替え                             │
└──────────────────────────────────────────────────────┘
                        │
                        │ ユーザーが"Correct Response"タブクリック
                        ▼
┌──────────────────────────────────────────────────────┐
│                  Correct状態                          │
│  - setTemplate('MAP_RESPONSE')                       │
│  - 即座にMap状態に遷移                                │
│  （ユーザーからは見えない - 一瞬で遷移）               │
└──────────────────────────────────────────────────────┘
                        │
                        │ 自動遷移
                        ▼
┌──────────────────────────────────────────────────────┐
│                    Map状態                            │
│  - 正答入力MathField表示                              │
│  - 別解追加・削除                                      │
│  - スコア設定                                          │
│  - Response Processing設定                            │
└──────────────────────────────────────────────────────┘
                        │
                        │ ユーザーが他のタブクリック
                        ▼
                  Question状態 or 他の状態
```

### データの共有

**全状態で共有**:
```javascript
this.widget.element  // interaction オブジェクト
    - interaction.prop(name)        // プロパティ取得
    - interaction.prop(name, value) // プロパティ設定
    - interaction.getResponseDeclaration()  // Response取得

this.widget.$container  // DOM コンテナ
this.widget.$form       // プロパティフォーム（Question状態）
this.widget.$responseForm  // レスポンスフォーム（Map状態）
```

**Question状態で設定 → Map状態で使用**:
```javascript
// Question状態で設定
interaction.prop('useGapExpression', true);
interaction.prop('tool_frac', true);
interaction.prop('tool_sqrt', false);

// Map状態で参照
if (this.inGapMode()) {  // useGapExpression を確認
    // Gap Expression用の処理
}
```

---

## まとめ

### 各状態の役割（要約）

| 状態 | ファイル | 行数 | 主な役割 | いつ使われるか |
|-----|---------|------|---------|-------------|
| **Widget** | Widget.js | 41行 | 状態管理の基盤 | インタラクション初期化時 |
| **Question** | Question.js | 500行 | プロパティ設定 | "Question"タブ選択時 |
| **Correct** | Correct.js | 35行 | Map状態への中継 | "Correct Response"タブ選択時（一瞬） |
| **Map** | Map.js | 700行 | 正答・採点設定 | Correct状態から自動遷移 / "Response Processing"タブ選択時 |

### 重要なポイント

1. **Question状態**: プロパティ・オプション・ツールボタンを設定
2. **Correct状態**: 実態はなく、Map状態への橋渡し
3. **Map状態**: 正答・別解・スコアという最も重要なデータを管理
4. **状態遷移**: TAOのタブUIと連動して自動的に遷移
5. **データ共有**: `this.widget.element`（interaction）を通じて状態間でデータ共有
6. **ライフサイクル**: 各状態は `create()/init()` で初期化、`exit()` でクリーンアップ

### 開発時の注意点

- **Question状態**: ツールボタン追加・削除時は `tools` オブジェクトと `propertiesForm.tpl` を両方編集
- **Map状態**: 正答保存ロジックは `saveAnswers()` に集約されている
- **状態遷移**: `this.widget.changeState('stateName')` で手動遷移可能
- **メモリリーク**: exit() でのクリーンアップを忘れずに（イベントリスナー、DOM、MathField等）

---

**最終更新**: 2026-02-02
