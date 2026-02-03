# jQuery・Lodash CVEベース セキュリティチェックリスト

## 対象ライブラリ
- **jQuery**: 2.1.1（2014年リリース、2016年EOL）
- **Lodash**: 4.17.21（2021年リリース）

## 最終更新日
2026-02-02

---

## jQuery 2.1.1 - CVEベース確認項目

### CVE-2015-9251: Cross-site Scripting (XSS) via AJAX

**CVSS**: 6.1 (Medium)
**影響**: jQuery < 3.0.0

#### 脆弱性の概要
クロスドメインAJAXリクエストで `dataType: "script"` を使用すると、攻撃者が任意のスクリプトを実行可能。

#### 確認項目

- [ ] **1-1.** `$.ajax()` でクロスドメインリクエストを行っていない
- [ ] **1-2.** `$.ajax()` で `dataType: "script"` を使用していない
- [ ] **1-3.** `$.getScript()` で外部URLを読み込んでいない
- [ ] **1-4.** JSONP（`dataType: "jsonp"`）を使用していない

#### 検索コマンド
```bash
# $.ajax 使用箇所を検索
grep -rn "\.ajax\(" --include="*.js"
grep -rn "dataType.*script" --include="*.js"
grep -rn "\.getScript" --include="*.js"
grep -rn "dataType.*jsonp" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **1-1.** ✅ `$.ajax()` 未使用
- [x] **1-2.** ✅ `dataType: "script"` 未使用
- [x] **1-3.** ✅ `$.getScript()` 未使用
- [x] **1-4.** ✅ JSONP 未使用

**リスク**: なし

---

### CVE-2019-11358: Prototype Pollution

**CVSS**: 6.1 (Medium)
**影響**: jQuery < 3.4.0

#### 脆弱性の概要
`$.extend(true, ...)` で `__proto__` や `constructor.prototype` を介してプロトタイプ汚染が可能。

#### 確認項目

- [ ] **2-1.** `$.extend()` にユーザー入力を含むオブジェクトを渡していない
- [ ] **2-2.** `$.extend(true, ...)` (deep extend) を使用していない
- [ ] **2-3.** `$.extend()` で外部データ（サーバーレスポンス、URLパラメータ等）をマージしていない
- [ ] **2-4.** `$.extend()` の第1引数が既存の重要なオブジェクトでない

#### 検索コマンド
```bash
# $.extend 使用箇所を検索
grep -rn "\.extend\(" --include="*.js"
grep -rn "\.extend(true" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **2-1.** ✅ `$.extend()` 未使用
- [x] **2-2.** ✅ deep extend 未使用
- [x] **2-3.** ✅ 外部データのマージなし
- [x] **2-4.** ✅ 該当なし

**リスク**: なし

---

### CVE-2020-11022: Cross-site Scripting (XSS) via htmlPrefilter

**CVSS**: 6.9 (Medium)
**影響**: jQuery < 3.5.0

#### 脆弱性の概要
HTML文字列のパース処理で `<option>` や `<style>` タグのサニタイズが不十分。

#### 確認項目

- [ ] **3-1.** `$()` コンストラクタにユーザー入力を含むHTML文字列を渡していない
- [ ] **3-2.** `$('<option>' + userInput + '</option>')` のようなコードがない
- [ ] **3-3.** `$('<style>' + userInput + '</style>')` のようなコードがない
- [ ] **3-4.** `.html()` にユーザー入力を含むHTML文字列を渡していない

#### 検索コマンド
```bash
# $() でHTML作成している箇所を検索
grep -rn "\$('<" --include="*.js"
grep -rn "\$(\"\`<" --include="*.js"
grep -rn "\.html\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **3-1.** ✅ ユーザー入力を含むHTML文字列なし
- [x] **3-2.** ✅ `<option>` タグ生成なし
- [x] **3-3.** ✅ `<style>` タグ生成なし
- [x] **3-4.** ✅ `.html()` は安全に使用（Handlebarsテンプレート経由）

**リスク**: なし

---

### CVE-2020-11023: Cross-site Scripting (XSS) via passing HTML

**CVSS**: 6.9 (Medium)
**影響**: jQuery < 3.5.0

#### 脆弱性の概要
特定のHTMLタグ（`<option>`, `<select>` 等）を含むHTML文字列のサニタイズが不十分。

#### 確認項目

- [ ] **4-1.** `.html()` にサーバーから受け取ったHTMLを直接渡していない
- [ ] **4-2.** `.append()/.prepend()` にHTML文字列+外部データを渡していない
- [ ] **4-3.** `.after()/.before()` にHTML文字列+外部データを渡していない
- [ ] **4-4.** 信頼できないソースからのHTML文字列を使用していない

#### 検索コマンド
```bash
# HTML挿入メソッドを検索
grep -rn "\.html\(" --include="*.js"
grep -rn "\.append\(" --include="*.js"
grep -rn "\.prepend\(" --include="*.js"
grep -rn "\.after\(" --include="*.js"
grep -rn "\.before\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **4-1.** ✅ サーバーHTMLの直接表示なし
- [x] **4-2.** ✅ HTML文字列+外部データなし（DOM要素のみ）
- [x] **4-3.** ✅ HTML文字列+外部データなし（DOM要素のみ）
- [x] **4-4.** ✅ 信頼できないHTMLソースなし

**リスク**: なし

---

## Lodash 4.17.21 - CVEベース確認項目

### CVE-2025-13465: Prototype Pollution via _.unset and _.omit

**CVSS**: 未公開（推定 Medium〜High）
**影響**: Lodash 4.0.0 〜 4.17.22
**修正**: Lodash 4.17.23+

#### 脆弱性の概要
`_.unset()` と `_.omit()` で `__proto__` を介してグローバルプロトタイプのメソッドを削除可能。

#### 確認項目

- [ ] **5-1.** `_.unset()` を使用していない
- [ ] **5-2.** `_.omit()` を使用していない
- [ ] **5-3.** `_.unset()` や `_.omit()` にユーザー入力を含むパスを渡していない
- [ ] **5-4.** 使用している場合、引数が完全に制御されている

#### 検索コマンド
```bash
# _.unset, _.omit 使用箇所を検索
grep -rn "_\.unset\(" --include="*.js"
grep -rn "_\.omit\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **5-1.** ✅ `_.unset()` 未使用
- [x] **5-2.** ✅ `_.omit()` 未使用
- [x] **5-3.** ✅ 該当なし
- [x] **5-4.** ✅ 該当なし

**リスク**: なし

**重要**: Lodash 4.17.21は本CVEに脆弱だが、影響を受けるメソッドを使用していないため実リスクなし。

---

### CVE-2019-10744: Prototype Pollution

**CVSS**: 9.1 (Critical)
**影響**: Lodash < 4.17.12
**修正**: Lodash 4.17.12+

#### 脆弱性の概要
`_.defaultsDeep()`, `_.merge()`, `_.zipObjectDeep()` で `__proto__` を介してプロトタイプ汚染が可能。

#### 確認項目

- [ ] **6-1.** `_.defaultsDeep()` を使用していない
- [ ] **6-2.** `_.merge()` を使用していない
- [ ] **6-3.** `_.zipObjectDeep()` を使用していない
- [ ] **6-4.** 使用している場合、ユーザー入力や外部データを渡していない

#### 検索コマンド
```bash
# Prototype Pollution関連メソッドを検索
grep -rn "_\.defaultsDeep\(" --include="*.js"
grep -rn "_\.merge\(" --include="*.js"
grep -rn "_\.zipObjectDeep\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **6-1.** ✅ `_.defaultsDeep()` 未使用
- [x] **6-2.** ✅ `_.merge()` 未使用
- [x] **6-3.** ✅ `_.zipObjectDeep()` 未使用
- [x] **6-4.** ✅ 該当なし

**リスク**: なし

**注**: Lodash 4.17.21では修正済み。

---

### CVE-2020-8203: Prototype Pollution

**CVSS**: 7.4 (High)
**影響**: Lodash < 4.17.19
**修正**: Lodash 4.17.19+

#### 脆弱性の概要
`_.zipObjectDeep()` で `__proto__` を介してプロトタイプ汚染が可能（CVE-2019-10744の亜種）。

#### 確認項目

- [ ] **7-1.** `_.zipObjectDeep()` を使用していない
- [ ] **7-2.** `_.zipObjectDeep()` にユーザー入力由来のキー配列を渡していない

#### 検索コマンド
```bash
grep -rn "_\.zipObjectDeep\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **7-1.** ✅ `_.zipObjectDeep()` 未使用
- [x] **7-2.** ✅ 該当なし

**リスク**: なし

**注**: Lodash 4.17.21では修正済み。

---

### CVE-2021-23337: Command Injection

**CVSS**: 7.2 (High)
**影響**: Lodash < 4.17.21
**修正**: Lodash 4.17.21+

#### 脆弱性の概要
`_.template()` でテンプレート文字列が適切にサニタイズされず、任意のコード実行が可能。

#### 確認項目

- [ ] **8-1.** `_.template()` を使用していない
- [ ] **8-2.** `_.template()` にユーザー入力を含むテンプレート文字列を渡していない
- [ ] **8-3.** `_.template()` の設定で `variable` オプションを使用している場合、適切に検証している

#### 検索コマンド
```bash
grep -rn "_\.template\(" --include="*.js"
```

#### 確認結果（mathEntryInteraction）
- [x] **8-1.** ✅ `_.template()` 未使用
- [x] **8-2.** ✅ 該当なし
- [x] **8-3.** ✅ 該当なし

**リスク**: なし

**注**: Lodash 4.17.21では修正済み。

---

## 補足: 安全なLodashメソッドの使用状況

### mathEntryInteractionで使用されているLodashメソッド

以下のメソッドは**CVEの影響を受けず、安全**:

| メソッド | 使用回数 | CVEリスク | 評価 |
|---------|---------|---------|------|
| `_.isArray()` | 3 | なし | ✅ 安全 |
| `_.parseInt()` | 2 | なし | ✅ 安全 |
| `_.keys()` | 2 | なし | ✅ 安全 |
| `_.assign()` | 2 | なし | ✅ 安全 |
| `_.toArray()` | 1 | なし | ✅ 安全 |
| `_.mapValues()` | 1 | なし | ✅ 安全 |
| `_.isUndefined()` | 1 | なし | ✅ 安全 |
| `_.isFunction()` | 1 | なし | ✅ 安全 |
| `_.isFinite()` | 1 | なし | ✅ 安全 |

**結論**: mathEntryInteractionで使用しているLodashメソッドは全て安全。

---

## チェックリスト実施記録

### jQuery 2.1.1

| CVE ID | 確認項目数 | 安全 | リスクあり | 実施日 | 担当者 |
|--------|----------|------|-----------|--------|-------|
| CVE-2015-9251 | 4 | ✅ 4 | 0 | 2026-02-02 | |
| CVE-2019-11358 | 4 | ✅ 4 | 0 | 2026-02-02 | |
| CVE-2020-11022 | 4 | ✅ 4 | 0 | 2026-02-02 | |
| CVE-2020-11023 | 4 | ✅ 4 | 0 | 2026-02-02 | |
| **合計** | **16** | **✅ 16** | **0** | | |

**総合評価**: ✅ 安全（全項目クリア）

### Lodash 4.17.21

| CVE ID | 確認項目数 | 安全 | リスクあり | 実施日 | 担当者 |
|--------|----------|------|-----------|--------|-------|
| CVE-2025-13465 ⚠️ | 4 | ✅ 4 | 0 | 2026-02-02 | |
| CVE-2019-10744 | 4 | ✅ 4 | 0 | 2026-02-02 | |
| CVE-2020-8203 | 2 | ✅ 2 | 0 | 2026-02-02 | |
| CVE-2021-23337 | 3 | ✅ 3 | 0 | 2026-02-02 | |
| **合計** | **13** | **✅ 13** | **0** | | |

**総合評価**: ✅ 安全（全項目クリア）

**注**: CVE-2025-13465は未パッチだが、影響を受けるメソッドを使用していないため実リスクなし。

---

## 最終結論

### mathEntryInteractionのCVEベース評価

| ライブラリ | バージョン | 既知CVE数 | 実リスク | 評価 |
|----------|----------|----------|---------|------|
| jQuery | 2.1.1 | 4件 | 0件 | ✅ 安全 |
| Lodash | 4.17.21 | 4件（1件未パッチ） | 0件 | ✅ 安全 |

**総合評価**: ✅ **安全**

**理由**:
1. **jQuery**: 4つの既知CVEがあるが、影響を受けるメソッド・使用パターンが全て不在
2. **Lodash**: 1つの未パッチCVE（CVE-2025-13465）があるが、脆弱なメソッド（`_.unset`, `_.omit`）を使用していない

### 推奨アクション

#### 短期（現状維持）
- ✅ CVEベースの確認は全てクリア
- ✅ 追加対応は不要

#### 中期（ライブラリアップグレード検討）
- [ ] jQuery 2.1.1 → 3.7.1 へのアップグレード（TAO Core対応待ち）
- [ ] Lodash 4.17.21 → 4.17.23 へのアップグレード（TAO Core対応待ち）

#### 長期（技術的負債解消）
- [ ] Lodashレス化（ネイティブJavaScript置き換え）
- [ ] jQueryレス化の検討（MathQuill依存により困難）

---

## 検索コマンド一覧（まとめ）

```bash
# jQuery関連
grep -rn "\.ajax\(" --include="*.js"
grep -rn "dataType.*script" --include="*.js"
grep -rn "\.getScript" --include="*.js"
grep -rn "dataType.*jsonp" --include="*.js"
grep -rn "\.extend\(" --include="*.js"
grep -rn "\.extend(true" --include="*.js"
grep -rn "\$('<" --include="*.js"
grep -rn "\.html\(" --include="*.js"
grep -rn "\.append\(" --include="*.js"
grep -rn "\.prepend\(" --include="*.js"
grep -rn "\.after\(" --include="*.js"
grep -rn "\.before\(" --include="*.js"

# Lodash関連
grep -rn "_\.unset\(" --include="*.js"
grep -rn "_\.omit\(" --include="*.js"
grep -rn "_\.defaultsDeep\(" --include="*.js"
grep -rn "_\.merge\(" --include="*.js"
grep -rn "_\.zipObjectDeep\(" --include="*.js"
grep -rn "_\.template\(" --include="*.js"
```

---

## 参考資料

### CVE詳細
- [CVE-2015-9251 (jQuery)](https://nvd.nist.gov/vuln/detail/CVE-2015-9251)
- [CVE-2019-11358 (jQuery)](https://nvd.nist.gov/vuln/detail/CVE-2019-11358)
- [CVE-2020-11022 (jQuery)](https://nvd.nist.gov/vuln/detail/CVE-2020-11022)
- [CVE-2020-11023 (jQuery)](https://nvd.nist.gov/vuln/detail/CVE-2020-11023)
- [CVE-2025-13465 (Lodash)](https://security.snyk.io/vuln/SNYK-JS-LODASH-15053838)
- [CVE-2019-10744 (Lodash)](https://nvd.nist.gov/vuln/detail/CVE-2019-10744)
- [CVE-2020-8203 (Lodash)](https://nvd.nist.gov/vuln/detail/CVE-2020-8203)
- [CVE-2021-23337 (Lodash)](https://nvd.nist.gov/vuln/detail/CVE-2021-23337)

### セキュリティガイド
- [OWASP Prototype Pollution](https://owasp.org/www-community/vulnerabilities/Prototype_Pollution)
- [jQuery Security](https://jquery.com/security/)
- [Snyk Lodash Vulnerabilities](https://security.snyk.io/package/npm/lodash)
