# jQuery XSSセキュリティチェックリスト

## 1. 禁止メソッド（ユーザー入力を含むHTML文字列での使用禁止）

### 高リスク - 常に禁止
- [ ] `.html(htmlString)` - innerHTML相当、ユーザー入力との組み合わせは禁止
- [ ] `$(htmlString)` - jQueryコンストラクタにユーザー入力を含むHTML文字列を渡すのは禁止

### 中リスク - HTML文字列での使用時のみ禁止
- [ ] `.append(htmlString)` - DOM要素または`.text()`で作成した要素のみ可
- [ ] `.prepend(htmlString)` - 同上
- [ ] `.after(htmlString)` - 同上
- [ ] `.before(htmlString)` - 同上
- [ ] `.replaceWith(htmlString)` - 同上
- [ ] `.wrap(htmlString)` - 同上
- [ ] `.wrapAll(htmlString)` - 同上
- [ ] `.wrapInner(htmlString)` - 同上

### その他
- [ ] `.attr('onclick', code)` - インラインイベントハンドラは禁止
- [ ] `.attr('href', 'javascript:...')` - javascript:スキームは禁止

## 2. 推奨される安全な代替方法

### テキスト表示
```javascript
// ❌ 禁止
$('#element').html(userInput);

// ✅ 推奨
$('#element').text(userInput);
element.textContent = userInput;
```

### フォーム値設定
```javascript
// ✅ 推奨
$('#input').val(userInput);
element.value = userInput;
```

### HTML要素の動的作成（ユーザー入力を含む場合）
```javascript
// ❌ 禁止
var html = '<div class="item">' + userInput + '</div>';
$('#container').append(html);

// ✅ 推奨パターン1: DOM要素として作成
var $element = $('<div>').addClass('item').text(userInput);
$('#container').append($element);

// ✅ 推奨パターン2: 静的テンプレート + text()
var $element = $('<div class="item"></div>');
$element.text(userInput);
$('#container').append($element);

// ✅ 推奨パターン3: ネイティブDOM
var element = document.createElement('div');
element.className = 'item';
element.textContent = userInput;
document.getElementById('container').appendChild(element);
```

### 属性設定
```javascript
// ✅ 推奨
$('#element').attr('data-value', userInput);
$('#element').attr('title', userInput);

// ❌ 禁止
$('#element').attr('onclick', 'doSomething("' + userInput + '")');
```

## 3. コードレビューチェック項目

### 静的解析
- [ ] `.html(` の全使用箇所を確認
- [ ] `.append(`, `.prepend(`, `.after(`, `.before(` でHTML文字列を渡している箇所を確認
- [ ] `$('<` でユーザー入力を含む箇所を確認
- [ ] 文字列連結 (`+`) または テンプレートリテラル (`` `${}` ``) でHTML生成している箇所を確認

### 動的解析
- [ ] ユーザー入力フィールドに `<script>alert('XSS')</script>` を入力してテスト
- [ ] ユーザー入力フィールドに `<img src=x onerror=alert('XSS')>` を入力してテスト
- [ ] ユーザー入力フィールドに `<svg onload=alert('XSS')>` を入力してテスト

## 4. mathEntryInteraction固有の確認箇所

### 確認すべきユーザー入力ポイント
- [ ] MathQuillフィールドの入力値 (`.latex()` の戻り値)
- [ ] Gapモードの各フィールド入力値
- [ ] プロパティパネルでの設定値（タイトル、ラベル等）
- [ ] サーバーから取得したresponse値の表示

### 確認すべき表示箇所
- [ ] Runtime: 数式フィールドの描画
- [ ] Runtime: Gap Expression各フィールドの描画
- [ ] Creator: プレビュー表示
- [ ] Creator: プロパティパネルでの値表示
- [ ] Creator: ツールチップ表示

## 5. 検索コマンド（grep/ripgrep）

```bash
# .html()の使用箇所を検索
grep -n "\.html(" runtime/*.js creator/*.js

# append/prepend等でHTML文字列を渡している可能性がある箇所
grep -n "\.append\|\.prepend\|\.after\|\.before" runtime/*.js creator/*.js

# 文字列連結でHTML生成している可能性
grep -n "'+.*<" runtime/*.js creator/*.js

# テンプレートリテラルでHTML生成している可能性
grep -n "\`.*<.*\${" runtime/*.js creator/*.js
```

## 6. 修正優先度

### P0 (即座に修正)
ユーザー入力を `.html()` や HTML文字列連結で直接表示している箇所

### P1 (早急に修正)
- サーバーから取得したデータを `.html()` で表示している箇所
- `.append()` 等でHTML文字列を渡している箇所

### P2 (計画的に修正)
- 静的コンテンツのみを `.html()` で表示している箇所（リファクタリング推奨）

## 7. 例外ケース（許可される場合）

以下の条件を**すべて**満たす場合のみ `.html()` 使用可:
1. ✅ 完全に静的なHTML（変数を含まない）
2. ✅ サーバーからのデータを含まない
3. ✅ ユーザー入力を含まない
4. ✅ コードレビューで承認済み

```javascript
// ✅ 例外的に許可（完全に静的）
$('#help').html('<strong>注意:</strong> 数式を入力してください');

// ❌ 禁止（サーバーデータ含む）
$('#message').html(response.message);

// ❌ 禁止（ユーザー入力含む）
$('#result').html('<div>' + userInput + '</div>');
```

## 8. レビュー実施記録

| 日付 | レビュー対象 | 担当者 | 結果 | 備考 |
|------|------------|--------|------|------|
| 2026-02-01 | 静的解析（全テンプレート） | Claude Code | ⚠️ 1件発見 | markup.tpl:2で`{{{prompt}}}`使用 |
| 2026-02-01 | 静的解析（jQuery使用箇所） | Claude Code | ✅ 問題なし | `.html()`は安全に使用されている |
| 2026-02-01 | 静的解析（append/prepend等） | Claude Code | ✅ 問題なし | DOM要素のみ渡している |

### レビュー詳細

#### 2026-02-01: テンプレートファイル静的解析

**検索コマンド**:
```bash
grep -rn "{{{" creator/tpl/
```

**結果**:
- **発見**: `markup.tpl:2` で `{{{prompt}}}` 使用
- **評価**: 中リスク（TAOのcontainerEditorのサニタイズに依存）
- **対応**: xss-review-findings.mdに詳細記載、追加調査が必要

#### 2026-02-01: jQuery `.html()` 使用箇所分析

**検索コマンド**:
```bash
grep -rn "\.html\(" --include="*.js"
```

**結果**:
- Map.js:132, Question.js:284 でHandlebarsテンプレート関数の戻り値を使用
- テンプレート内で非エスケープ出力（`{{{}}}`）は markup.tpl:2 のみ
- **評価**: 現時点で明確なXSS脆弱性なし

#### 2026-02-01: jQuery `.append()/.prepend()/.after()/.before()` 使用箇所分析

**検索コマンド**:
```bash
grep -rn "\.(append|prepend|after|before|replaceWith)\(" --include="*.js"
```

**結果**:
- 全ての使用箇所でDOM要素またはテンプレート関数戻り値を使用
- HTML文字列+ユーザー入力の直接連結はなし
- **評価**: 安全

## 9. 参考資料

- OWASP XSS Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- jQuery API Documentation: https://api.jquery.com/
- DOM XSS Wiki: https://github.com/wisec/domxsswiki/wiki/jQuery
