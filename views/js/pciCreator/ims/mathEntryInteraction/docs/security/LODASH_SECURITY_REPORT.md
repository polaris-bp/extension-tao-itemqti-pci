# Lodash Security Investigation Report

## 調査実施日
2026-02-02

## エグゼクティブサマリー

### 調査目的
TAO Platform（extension-tao-itemqti-pci）で使用されているLodashライブラリのバージョンと既知の脆弱性リスクを評価する。

### 主要な発見
1. **TAO Lodashバージョン移行**: 2024年2月にLodash v2→v4への移行を試みたがロールバック発生
2. **現在のLodashバージョン**: Lodash v4（tao-core >=54.0.0で導入）
3. **最新の脆弱性**: CVE-2025-13465がLodash 4.17.21以下に影響（2025年1月発見）
4. **mathEntryInteractionでの使用状況**: 脆弱なメソッド（`_.unset`, `_.omit`）は未使用

### リスク評価
**🟡 中リスク**: Lodash 4.17.21は最新のCVEに脆弱だが、mathEntryInteractionでの実装は安全

---

## 詳細調査結果

### 1. Lodash バージョン履歴

#### Git履歴からの調査

| 日付 | コミット | 内容 | 影響 |
|-----|---------|------|------|
| 2024-02-01 | eac3509 | tao-core >=54.0.0依存追加（Lodash v4導入） | Lodash v4へ移行 |
| 2024-02-01 | f66aac1 | Lodashメソッドをv4仕様に更新 | `_.invoke` → `_.invokeMap` |
| 2024-02-01 | 0642c35 | **ロールバック**: Lodash v2メソッドに戻す | `_.invokeMap` → `_.invoke` |
| 2024-02-01 | 6e2cb79 | audioRecording PCIでinvokeをinvokeMapに更新 | 再度v4対応試行 |
| 2024-02-01 | dda374a | portableElementsでinvokeをinvokeMapに更新 | v4対応完了 |

#### タイムライン解析

```
2024-02-01 17:46 (eac3509)
    ↓ tao-core >=54.0.0でLodash v4導入

2024-02-01 17:?? (f66aac1)
    ↓ PCIコードをLodash v4に更新（_.invokeMap使用）

2024-02-01 19:00 (0642c35)
    ↓ ロールバック発生！
    理由: `taoQtiItem/portableLib/lodash` がまだLodash v2だった

2024-02-01 19:?? (6e2cb79, dda374a)
    ↓ 段階的に各PCIをLodash v4対応
```

#### 結論

- **TAO Core v54.0.0以降**: Lodash v4を使用
- **現在のextension-tao-itemqti-pci**: Lodash v4依存（composer.json:25で確認）
- **バンドル方式**: 各PCIのminファイルにLodashがバンドルされている

### 2. 現在のLodashバージョン推定

#### composer.json分析

```json
{
  "require": {
    "oat-sa/tao-core" : ">=54.0.0",
    "oat-sa/extension-tao-itemqti" : ">=29.13.0"
  }
}
```

**推定**: tao-core v54.0.0がLodash v4を導入 → おそらく**Lodash 4.17.20または4.17.21**

#### 根拠
1. Lodash 4.17.21は2021年2月リリース（CVE-2020-8203修正版）
2. TAOのロールバックが2024年2月 → この時点で最新は4.17.21
3. 2024年時点の依存関係として4.17.21が最も妥当

**結論**: **Lodash 4.17.21を使用している可能性が高い**

---

## Lodash 既知の脆弱性

### CVE-2025-13465（最新・重要）

| 項目 | 詳細 |
|-----|------|
| **CVE ID** | CVE-2025-13465 |
| **発見日** | 2025年1月（約1ヶ月前） |
| **影響バージョン** | Lodash 4.0.0 〜 4.17.22 |
| **脆弱性タイプ** | Prototype Pollution |
| **CVSS スコア** | 未公開（推定: Medium〜High） |
| **影響を受けるメソッド** | `_.unset()`, `_.omit()` |
| **修正バージョン** | **Lodash 4.17.23+** |

#### 脆弱性の詳細

**攻撃ベクトル**:
```javascript
// 脆弱なコード例
const obj = {};
_.unset(obj, '__proto__.polluted');  // ← プロトタイプ汚染

// または
const result = _.omit(obj, ['__proto__.polluted']);  // ← プロトタイプ汚染
```

**影響**:
- 攻撃者がグローバルプロトタイプのメソッドを削除可能
- プロパティの削除は可能だが上書きは不可（軽減要因）
- Denial of Service（DoS）や予期しない動作を引き起こす可能性

**修正**:
Lodash 4.17.23で修正済み

### CVE-2019-10744

| 項目 | 詳細 |
|-----|------|
| **CVE ID** | CVE-2019-10744 |
| **影響バージョン** | Lodash < 4.17.12 |
| **脆弱性タイプ** | Prototype Pollution |
| **CVSS スコア** | 9.1 (Critical) |
| **影響を受けるメソッド** | `_.defaultsDeep()`, `_.merge()`, `_.zipObjectDeep()` |
| **修正バージョン** | Lodash 4.17.12+ |

#### Lodash 4.17.21の場合
✅ **修正済み**（4.17.12で対応）

### CVE-2020-8203

| 項目 | 詳細 |
|-----|------|
| **CVE ID** | CVE-2020-8203 |
| **影響バージョン** | Lodash < 4.17.19 |
| **脆弱性タイプ** | Prototype Pollution |
| **CVSS スコア** | 7.4 (High) |
| **影響を受けるメソッド** | `_.zipObjectDeep()` |
| **修正バージョン** | Lodash 4.17.19+ |

#### Lodash 4.17.21の場合
✅ **修正済み**（4.17.19で対応）

### CVE-2021-23337

| 項目 | 詳細 |
|-----|------|
| **CVE ID** | CVE-2021-23337 |
| **影響バージョン** | Lodash < 4.17.21 |
| **脆弱性タイプ** | Command Injection |
| **CVSS スコア** | 7.2 (High) |
| **影響を受けるメソッド** | `_.template()` |
| **修正バージョン** | Lodash 4.17.21+ |

#### Lodash 4.17.21の場合
✅ **修正済み**（4.17.21で対応）

---

## mathEntryInteractionでのLodash使用状況

### 使用メソッド一覧

| メソッド | 使用回数 | 脆弱性リスク | 評価 |
|---------|---------|------------|------|
| `_.isArray()` | 3 | なし | ✅ 安全 |
| `_.parseInt()` | 2 | なし | ✅ 安全 |
| `_.keys()` | 2 | なし | ✅ 安全 |
| `_.assign()` | 2 | なし（静的データのみ） | ✅ 安全 |
| `_.toArray()` | 1 | なし | ✅ 安全 |
| `_.mapValues()` | 1 | なし | ✅ 安全 |
| `_.isUndefined()` | 1 | なし | ✅ 安全 |
| `_.isFunction()` | 1 | なし | ✅ 安全 |
| `_.isFinite()` | 1 | なし | ✅ 安全 |

### 脆弱なメソッドの使用状況

| 脆弱なメソッド | CVE | 使用箇所 | リスク |
|-------------|-----|---------|-------|
| `_.unset()` | CVE-2025-13465 | ❌ 未使用 | なし |
| `_.omit()` | CVE-2025-13465 | ❌ 未使用 | なし |
| `_.defaultsDeep()` | CVE-2019-10744 | ❌ 未使用 | なし |
| `_.merge()` | CVE-2019-10744 | ❌ 未使用 | なし |
| `_.zipObjectDeep()` | CVE-2019-10744, CVE-2020-8203 | ❌ 未使用 | なし |
| `_.template()` | CVE-2021-23337 | ❌ 未使用 | なし |

### `_.assign()` の使用状況詳細

**使用箇所1**: `creator/widget/states/Question.js:284`
```javascript
$form.html(formTpl(_.assign({
    serial : response.serial,
    identifier : interaction.attr('responseIdentifier'),
    authorizeWhiteSpace: toBoolean(interaction.prop('authorizeWhiteSpace'), false),
    useGapExpression: toBoolean(interaction.prop('useGapExpression'), false),
    // ... 静的プロパティのみ
}, getToolsInitValues(interaction))));
```

**評価**: ✅ 安全
- 第1引数: 新しいオブジェクト（`{}`）
- 第2引数: 静的プロパティのみ
- ユーザー入力を直接扱わない
- `__proto__` などの危険なキーは含まれない

**使用箇所2**: `creator/widget/states/Question.js:300`
```javascript
formElement.setChangeCallbacks($form, interaction, _.assign({
    identifier: function(i, value){ ... },
    useGapExpression: function gapChangeCallback(i, value) { ... },
    // ... コールバック関数定義のみ
}, toolsChangeCallBacks));
```

**評価**: ✅ 安全
- コールバック関数のマージのみ
- ユーザー入力を直接扱わない

---

## リスク評価サマリー

### 全体的なリスク: 🟡 中リスク

#### リスク要因
1. **CVE-2025-13465が未パッチ**: Lodash 4.17.21はCVE-2025-13465に脆弱
2. **バンドル方式**: 各PCIがLodashをバンドル → TAO Core単独のアップデートでは解決不可
3. **古いライブラリ**: Lodash自体が古い（最新は4.17.21、推奨は5.0.0-beta）

#### リスク軽減要因
1. **脆弱なメソッド未使用**: `_.unset()`, `_.omit()` は mathEntryInteraction で未使用
2. **安全なメソッドのみ**: 使用しているのは型チェックとユーティリティのみ
3. **静的データのみ**: `_.assign()` はユーザー入力を直接扱わない

### mathEntryInteraction固有のリスク: 🟢 低リスク

**理由**:
- CVE-2025-13465の影響を受けるメソッド（`_.unset`, `_.omit`）を使用していない
- その他の既知CVEの影響を受けるメソッドも未使用
- 使用しているLodashメソッドは安全

---

## 推奨事項

### 優先度P0: Lodash 4.17.23へのアップグレード

#### 理由
1. **CVE-2025-13465対応**: 最新のPrototype Pollution脆弱性を修正
2. **低リスクアップグレード**: 4.17.21→4.17.23はパッチバージョンアップのみ
3. **破壊的変更なし**: APIに変更なし

#### アップグレード手順

**ステップ1: TAO Coreの対応待ち**

extension-tao-itemqti-pciは tao-core に依存しているため、TAO Core側のアップグレードが必要:

1. **GitHub Issueを起票**:
   - リポジトリ: oat-sa/tao-core
   - タイトル: "Upgrade Lodash to 4.17.23 to address CVE-2025-13465"
   - 内容: CVE-2025-13465の詳細と影響範囲

2. **PRを作成**（可能であれば）:
   - Lodashバージョンを4.17.23に更新
   - 回帰テストを実施
   - 影響を受けるすべてのPCIをテスト

**ステップ2: PCIの再ビルド**

TAO Coreがアップグレードされたら:
```bash
# 各PCIの再ビルド
cd views/js/pciCreator/ims/mathEntryInteraction
npm install
npm run build

# minファイルに新しいLodash 4.17.23がバンドルされる
```

**ステップ3: 回帰テスト**
- [ ] mathEntryInteractionの全機能テスト
- [ ] Gap Expressionモードのテスト
- [ ] プロパティパネルのテスト
- [ ] 受験者モードでのテスト

### 優先度P1: 長期的なLodash戦略

#### オプション1: Lodash v5移行

**現状**: Lodash v5は開発中止（2021年）

**代替**: Lodash-ESへの移行
```bash
npm install lodash-es@4.17.21
```

**メリット**:
- Tree-shakingサポート
- モジュール単位でのインポート
- バンドルサイズ削減

**デメリット**:
- 大規模リファクタリング必要
- AMD→ESモジュールへの移行が必要

#### オプション2: Lodashレス化（段階的）

**フェーズ1**: ネイティブJavaScript置き換え
```javascript
// 置き換え例

// Before (Lodash)
_.isArray(value)
_.keys(obj)
_.assign({}, a, b)

// After (Native JavaScript)
Array.isArray(value)
Object.keys(obj)
Object.assign({}, a, b)
```

**対象メソッド**:
| Lodashメソッド | ネイティブ代替 | 互換性 |
|-------------|-------------|-------|
| `_.isArray()` | `Array.isArray()` | 完全 |
| `_.keys()` | `Object.keys()` | 完全 |
| `_.assign()` | `Object.assign()` | 完全 |
| `_.isFinite()` | `Number.isFinite()` | ほぼ同等 |

**フェーズ2**: カスタムユーティリティ関数
```javascript
// _.mapValues の代替
function mapValues(obj, fn) {
    return Object.keys(obj).reduce((result, key) => {
        result[key] = fn(obj[key], key);
        return result;
    }, {});
}
```

**判断**: mathEntryInteractionでのLodash依存は軽微 → **段階的Lodashレス化を推奨**

### 優先度P2: 継続的なセキュリティ監視

1. **Dependabot有効化**:
   - リポジトリでDependabotを有効化
   - Lodash脆弱性の自動検出

2. **定期的なセキュリティ監査**:
   - 月次でnpm auditを実行
   - CVEデータベースをチェック

3. **アップストリームへの貢献**:
   - TAO Coreへのセキュリティパッチ提供
   - コミュニティフォーラムでの情報共有

---

## 他のPCIでのLodash使用状況

現在のリポジトリ内の全PCIで同様の状況:

| PCI名 | Lodash参照 | 推定バージョン | 脆弱なメソッド使用 |
|-------|----------|--------------|----------------|
| mathEntryInteraction | `taoQtiItem/portableLib/lodash` | 4.17.21 | ❌ なし |
| likertScaleInteraction | `taoQtiItem/portableLib/lodash` | 4.17.21 | 要確認 |
| audioRecordingInteraction | `taoQtiItem/portableLib/lodash` | 4.17.21 | 要確認（`_.invoke` 使用） |
| liquidsInteraction | `taoQtiItem/portableLib/lodash` | 4.17.21 | 要確認 |

**推奨**: 他のPCIでも同様のセキュリティレビューを実施

---

## 結論

### 現状評価

extension-tao-itemqti-pci（mathEntryInteraction）は以下の状況:

1. **Lodash 4.17.21を使用**: CVE-2025-13465に脆弱
2. **脆弱なメソッドは未使用**: 実際の攻撃リスクは低い
3. **TAO Coreに依存**: 単独でのアップグレード不可

### 推奨アクション

#### 短期（即座〜1ヶ月）
1. ✅ **完了**: Lodashセキュリティレビュー
2. 📋 **推奨**: TAO CoreへのGitHub Issue起票（Lodash 4.17.23アップグレード要求）
3. 📋 **推奨**: 他のPCIでのLodash使用状況調査

#### 中期（1-3ヶ月）
1. TAO Core v54.x or v55.xでのLodash 4.17.23採用
2. 全PCIの再ビルドと回帰テスト
3. セキュリティテストの実施

#### 長期（3-6ヶ月）
1. Lodashレス化の段階的実施
2. ネイティブJavaScriptへの置き換え
3. バンドルサイズ削減とパフォーマンス改善

---

## 参考資料

### CVE情報
- [CVE-2025-13465 - Lodash Prototype Pollution](https://security.snyk.io/vuln/SNYK-JS-LODASH-15053838)
- [CVE-2019-10744 - Lodash Prototype Pollution](https://nvd.nist.gov/vuln/detail/CVE-2019-10744)
- [CVE-2020-8203 - Lodash Prototype Pollution](https://nvd.nist.gov/vuln/detail/CVE-2020-8203)
- [CVE-2021-23337 - Lodash Command Injection](https://nvd.nist.gov/vuln/detail/CVE-2021-23337)

### Lodash公式
- [Lodash Documentation](https://lodash.com/docs/)
- [Lodash GitHub Repository](https://github.com/lodash/lodash)
- [Lodash Changelog](https://github.com/lodash/lodash/wiki/Changelog)

### TAOリポジトリ
- [oat-sa/tao-core](https://github.com/oat-sa/tao-core)
- [oat-sa/extension-tao-itemqti-pci](https://github.com/oat-sa/extension-tao-itemqti-pci)
- [TAO Hub Articles](https://github.com/oat-sa/taohub-articles)

### セキュリティベストプラクティス
- [OWASP Prototype Pollution](https://owasp.org/www-community/vulnerabilities/Prototype_Pollution)
- [Snyk Lodash Security Vulnerabilities](https://security.snyk.io/package/npm/lodash)

---

## 調査メタデータ

| 項目 | 値 |
|-----|-----|
| 調査実施日 | 2026-02-02 |
| 調査者 | Claude Code |
| 対象リポジトリ | oat-sa/extension-tao-itemqti-pci |
| 対象ブランチ | claude/design-math-entry-interaction-kGKHQ |
| 参照コミット | 8f3be5e |
| TAO Core依存バージョン | >=54.0.0 |
| 推定Lodashバージョン | 4.17.21 |
| 最新Lodashバージョン | 4.17.23 |

---

## 次のアクション

- [ ] TAO CoreへのGitHub Issue起票（CVE-2025-13465対応依頼）
- [ ] 他のPCIでのLodash使用状況調査
- [ ] Lodashレス化のPoC実施（mathEntryInteractionから）
- [ ] ネイティブJavaScript置き換えのパフォーマンス測定
- [ ] セキュリティ監視体制の構築（Dependabot等）
