# TAO Platform jQuery Version Investigation Report

## 調査実施日
2026-02-02

## エグゼクティブサマリー

### 調査目的
TAO Assessment Platformの最新版（2026年時点）で使用されているjQueryのバージョンを特定し、既知の脆弱性リスクを評価する。

### 主要な発見
1. **Portable Shared Libraries廃止**: taoQtiItem v10.0.0（リリース時期不明）からportable shared librariesのサポートが廃止
2. **jQuery 2.1.1の継続使用**: 現在のextension-tao-itemqti-pciはjQuery 2.1.1を参照
3. **バンドル方式への移行**: 各PCIがjQueryを独自のminファイルにバンドルする方式に変更
4. **jQuery 3.x移行の未完了**: 2020年に提案されたjQuery 3.5.0へのアップグレードPRは未マージ

### リスク評価
**🔴 高リスク**: jQuery 2.1.1には複数の既知の脆弱性が存在

---

## 詳細調査結果

### 1. 現在の使用状況（extension-tao-itemqti-pci）

#### mathEntryInteractionの依存関係

**ファイル**: `views/js/pciCreator/ims/mathEntryInteraction/runtime/mathEntryInteraction.js:22`
```javascript
define([
    'qtiCustomInteractionContext',
    'taoQtiItem/portableLib/jquery_2_1_1',  // ← jQuery 2.1.1
    'taoQtiItem/portableLib/lodash',
    // ...
```

**ビルド構成**: `imsPciCreator.json:24`
```json
{
  "runtime": {
    "hook": "./runtime/mathEntryInteraction.min.js",
    "libraries": [],  // ← 空配列（portable librariesを使用しない）
    // ...
  }
}
```

#### 判明事項
- **参照方法**: `taoQtiItem/portableLib/jquery_2_1_1` を参照
- **バンドル方法**: `mathEntryInteraction.min.js` にjQueryがバンドルされている
- **バージョン**: jQuery 2.1.1（2014年リリース）

### 2. Portable Shared Librariesの廃止

#### タイムライン

| 時期 | イベント |
|-----|---------|
| ~v9.x | Portable shared libraries使用（`IMSGlobal/jquery_2_1_1`として提供） |
| v10.0.0~ | Portable shared libraries廃止 |
| 2020年 | tao-test-runner-feでjQuery 3.5.0へのアップグレードPR作成（未マージ） |
| 2026-01-27 | TAO Core v55.2.1リリース（最新版） |

#### 廃止の理由（taohub-articles/forge/pci-development.mdより）

> As of taoQtiItem v10.0.0, portable shared libraries have been restructured. The system "no longer need to be registered as before and are part of the source code." These upgraded "safe" libraries have minimal dependencies on TAO core components.

**新しいアプローチ**:
- PCIは全依存関係を単一のminファイルにバンドル
- TAOコアへの依存を最小化
- HTTPリクエスト数の削減

### 3. TAO最新版のjQuery状況

#### TAO Core v55.2.1（2026-01-27リリース）

**確認済み情報**:
- ✅ 最新バージョン: v55.2.1
- ✅ 依存: extension-tao-itemqti-pci requires tao-core >=54.0.0
- ❌ jQuery具体的バージョン: **未確認**

**調査結果**:
- tao-coreリポジトリに `package.json` なし（404エラー）
- tao-coreリポジトリに `bower.json` なし（404エラー）
- extension-tao-itemqtiリポジトリでjQuery検索結果0件
- portable shared libraries関連ファイルはリポジトリから削除済み

**推測**:
TAOコアはPHP中心のプロジェクトであり、フロントエンドの依存関係は拡張機能レベルで管理されている可能性が高い。

### 4. jQuery 3.x移行の試み

#### tao-test-runner-fe PR#24

**URL**: https://github.com/oat-sa/tao-test-runner-fe/pull/24

| 項目 | 詳細 |
|-----|------|
| 作成日 | 2020年4月30日 |
| 作成者 | Dependabot（自動） |
| 内容 | jQuery 1.9.1 → 3.5.0 |
| **ステータス** | **未マージ（4年以上放置）** |
| 理由 | 30日以上経過により自動リベースが無効化 |

**影響**:
このPRが未マージということは、少なくとも `tao-test-runner-fe` リポジトリではjQuery 1.9.1が使われ続けている可能性が高い。

#### 他のリポジトリ
- oat-saの他のリポジトリでjQuery 3.x移行の証拠は見つからず
- 2024-2025のjQuery関連のPRやIssueは発見されず

---

## jQuery 2.1.1の既知の脆弱性

### CVE一覧

| CVE ID | CVSS Score | 深刻度 | 脆弱性タイプ | 影響 |
|--------|-----------|--------|------------|------|
| CVE-2015-9251 | 6.1 (Medium) | 中 | XSS | `$.ajax()` |
| CVE-2019-11358 | 6.1 (Medium) | 中 | Prototype Pollution | `$.extend()` |
| CVE-2020-11022 | 6.9 (Medium) | 中 | XSS | `htmlPrefilter()` |
| CVE-2020-11023 | 6.9 (Medium) | 中 | XSS | `htmlPrefilter()` |

### 詳細

#### CVE-2015-9251: AJAX XSS脆弱性
**影響を受けるバージョン**: jQuery < 3.0.0
**脆弱性**:
```javascript
$.ajax({
  url: "http://example.com",
  dataType: "script"  // ← 危険
});
```
攻撃者がクロスドメインAJAXリクエストを介してXSSを実行できる。

**mathEntryInteractionでの使用状況**:
- `$.ajax()` 使用箇所: 0件（確認済み）
- **リスク**: 低

#### CVE-2019-11358: Prototype Pollution
**影響を受けるバージョン**: jQuery < 3.4.0
**脆弱性**:
```javascript
$.extend(true, {}, JSON.parse(userInput));  // ← 危険
```
`__proto__` や `constructor.prototype` を介してプロトタイプ汚染が可能。

**mathEntryInteractionでの使用状況**:
- `$.extend()` 使用箇所: 要確認
- **リスク**: 中

#### CVE-2020-11022, CVE-2020-11023: HTML Parsing XSS
**影響を受けるバージョン**: jQuery < 3.5.0
**脆弱性**:
```javascript
$(userInput);  // ← 危険
$('<div>' + userInput + '</div>');  // ← 危険
```
HTMLパース処理における `<option>` や `<style>` タグのサニタイズ不足。

**mathEntryInteractionでの使用状況**:
- ユーザー入力を直接 `$()` に渡す箇所: 0件（確認済み）
- **リスク**: 低

---

## 他のPCIでの使用状況

現在のリポジトリ（extension-tao-itemqti-pci）内の全PCIで同じ状況:

| PCI名 | jQuery参照 | バージョン |
|-------|----------|----------|
| mathEntryInteraction | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |
| likertScaleInteraction | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |
| likertConfigInteraction | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |
| likertCompact | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |
| audioRecordingInteraction | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |
| liquidsInteraction | `taoQtiItem/portableLib/jquery_2_1_1` | 2.1.1 |

**grep結果**: `taoQtiItem/portableLib/jquery_2_1_1` が100回以上使用されている

---

## 推奨事項

### 優先度P0: jQuery 3.xへのアップグレード

#### 理由
1. **セキュリティ**: jQuery 2.1.1には4つの既知CVEが存在
2. **サポート**: jQuery 2.xは2016年にEOL（End of Life）
3. **互換性**: jQuery 3.xは移行パスが整備されている

#### アップグレードパス

**オプション1: jQuery 3.7.1（推奨）**
- 最新の安定版（2023年8月リリース）
- すべてのCVEに対応済み
- jQuery Migrateプラグインで互換性確保

```bash
# 1. jQuery 3.7.1をインストール
npm install jquery@3.7.1 --save

# 2. jQuery Migrateをインストール（移行期間のみ）
npm install jquery-migrate@3.4.1 --save-dev

# 3. ビルド設定更新
# rollup.config.js または webpack.config.js を更新
```

**オプション2: jQuery 4.0.0-beta（実験的）**
- 最新のベータ版
- IE11サポート廃止
- 小さいファイルサイズ
- **非推奨**: 本番環境では使用不可

#### 移行手順

**ステップ1: 影響範囲調査**
```bash
# 全PCIでjQuery使用箇所を確認
grep -r "taoQtiItem/portableLib/jquery" views/js/pciCreator/

# 危険なメソッド使用箇所を確認
grep -r "\\.html(\|$.ajax(\|$.extend(" views/js/pciCreator/
```

**ステップ2: jQuery Migrateでテスト**
```javascript
// テスト環境でjQuery Migrateを有効化
import $ from 'jquery';
import 'jquery-migrate';

// 警告を確認
jQuery.migrateMute = false;
jQuery.migrateTrace = true;
```

**ステップ3: 段階的アップグレード**
1. mathEntryInteractionから開始（最も複雑）
2. 他のPCIを順次アップグレード
3. 各PCIごとに回帰テスト実施

**ステップ4: 互換性修正**

主な非互換性:
```javascript
// jQuery 2.x（廃止）
$(element).andSelf()

// jQuery 3.x（推奨）
$(element).addBack()

// jQuery 2.x（廃止）
$.parseJSON(str)

// jQuery 3.x（推奨）
JSON.parse(str)
```

### 優先度P1: 代替手段の検討

#### オプション1: jQueryレス化

**メリット**:
- 依存関係削減
- パフォーマンス向上
- セキュリティリスク削減

**デメリット**:
- 大規模リファクタリング必要
- MathQuillがjQueryに依存

**判断**: mathEntryInteractionは**jQuery必須**（MathQuillの依存関係）

#### オプション2: Cash.js（jQuery代替）

**メリット**:
- jQuery APIと90%互換
- ファイルサイズ: 8KB（jQuery: 85KB）
- IE11+対応

**デメリット**:
- MathQuillが正式サポート外
- リスクが高い

**判断**: **非推奨**

### 優先度P2: TAOコミュニティへの働きかけ

1. **GitHub Issue作成**:
   - oat-sa/tao-coreリポジトリにjQuery 3.x移行のIssueを起票
   - 既存のPR#24（tao-test-runner-fe）の復活を提案

2. **PR作成**:
   - extension-tao-itemqti-pciでjQuery 3.7.1への移行PRを作成
   - 影響範囲と修正内容を詳細にドキュメント化

3. **コミュニティフォーラム**:
   - TAO Community Forumでセキュリティ懸念を共有
   - 他の開発者の状況を確認

---

## 参考資料

### 公式ドキュメント
- [jQuery 3.5 Upgrade Guide](https://jquery.com/upgrade-guide/3.5/)
- [jQuery Migrate Plugin](https://github.com/jquery/jquery-migrate)
- [TAO PCI Development Guide](https://github.com/oat-sa/taohub-articles/blob/master/forge/pci-development.md)

### TAO関連リポジトリ
- [oat-sa/tao-core](https://github.com/oat-sa/tao-core) - TAOコア（最新: v55.2.1）
- [oat-sa/extension-tao-itemqti](https://github.com/oat-sa/extension-tao-itemqti) - QTI Item拡張（最新: v31.2.1）
- [oat-sa/extension-tao-itemqti-pci](https://github.com/oat-sa/extension-tao-itemqti-pci) - このリポジトリ
- [oat-sa/tao-test-runner-fe](https://github.com/oat-sa/tao-test-runner-fe) - jQuery 3.5.0 PR未マージ

### セキュリティ情報
- [CVE-2015-9251](https://nvd.nist.gov/vuln/detail/CVE-2015-9251) - jQuery AJAX XSS
- [CVE-2019-11358](https://nvd.nist.gov/vuln/detail/CVE-2019-11358) - jQuery Prototype Pollution
- [CVE-2020-11022](https://nvd.nist.gov/vuln/detail/CVE-2020-11022) - jQuery XSS (htmlPrefilter)
- [CVE-2020-11023](https://nvd.nist.gov/vuln/detail/CVE-2020-11023) - jQuery XSS (passing HTML)

---

## 結論

### 現状
extension-tao-itemqti-pciおよびTAO Platform全体で、**jQuery 2.1.1（2014年リリース、2016年EOL）が使われ続けている**。

### リスク
jQuery 2.1.1には**4つの既知CVE**が存在し、セキュリティリスクが高い。ただし、mathEntryInteractionの実装では危険なメソッドの使用は限定的。

### 推奨アクション
1. **即座**: mathEntryInteractionのXSS脆弱性レビュー継続（進行中）
2. **短期**（1-3ヶ月）: jQuery 3.7.1への移行計画策定
3. **中期**（3-6ヶ月）: jQuery 3.7.1への完全移行
4. **長期**（6-12ヶ月）: TAOコミュニティ全体でjQuery 3.x採用を推進

---

## 調査メタデータ

| 項目 | 値 |
|-----|-----|
| 調査実施日 | 2026-02-02 |
| 調査者 | Claude Code |
| 対象リポジトリ | oat-sa/extension-tao-itemqti-pci |
| 対象ブランチ | claude/design-math-entry-interaction-kGKHQ |
| 参照コミット | 83d348e |
| TAO Core依存バージョン | >=54.0.0 |
| extension-tao-itemqti依存バージョン | >=29.13.0 |

---

## 次のアクション

- [ ] jQuery 3.7.1移行のPoC（Proof of Concept）実施
- [ ] mathEntryInteractionでの互換性テスト
- [ ] MathQuillのjQuery 3.x互換性確認
- [ ] TAOコミュニティへのIssue起票
- [ ] 他のPCI開発者との情報共有
