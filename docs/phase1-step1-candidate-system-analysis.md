# Phase 1 - Step 1: 現在の候補取得システムの詳細調査と設計検討

**調査完了日**: 2024-11-08
**調査対象**: azooKey-Desktop 変換候補取得システム
**目的**: 予測変換機能追加のための設計基盤確立

---

## エグゼクティブサマリー

### 調査結果の概要

azooKey-Desktop の変換候補取得システムは、**よく設計された実装**であり、予測変換機能の統合は**技術的に実現可能**で、実装の複雑性も**低い**と判断されます。

### 主要な発見

1. **予測機能の API は既に存在するが無効化されている**
   - `requireJapanesePrediction: false`（現在無効）
   - `requireEnglishPrediction: false`（現在無効）
   - これらを `true` に変更することで予測機能が有効化される可能性

2. **ConversionResult に予測候補を統合可能**
   - mainResults と firstClauseResults が既に区分
   - predictionResults フィールドが存在する可能性（要確認）

3. **実装の変更量は最小限**
   - Phase 1（基本統合）: 約 15 行の変更
   - Phase 2（UI 表示）: 約 30-35 行の追加
   - 全体: 145-250 行の追加/修正

### 推奨アプローチ: ハイブリッド方式

**基本戦略**:
1. ConversionResult に predictionResults フィールドを利用
2. updateRawCandidate() で `requireJapanesePrediction = true` に変更
3. candidates プロパティで予測候補を統合
4. getCurrentCandidateWindow() で表示制御

**見積もり期間**: 10-14 週間（段階的実装）

### 即座に実施すべきアクション

**高優先度（1 週目）**:
- [ ] AzooKeyKanaKanjiConverter の API ドキュメント確認
  - predictionResults の存在確認
  - requireJapanesePrediction の実装確認
- [ ] Prototype A（options() のみ変更）のテスト実施

---

## 1. システムアーキテクチャ分析

### 1.1 モジュール構成

```
┌─────────────────────────────────────────────────────────────────┐
│                      ユーザ入力イベント                            │
│              (ClientAction via IMK Framework)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │ azooKeyMacInputController      │
        │ handleClientAction()            │
        └────────────┬───────────────────┘
                     │
              ┌──────▼──────┐
              │ 何をするか?   │
              └──────┬──────┘
                     │
        ┌────────────┴───────────┬──────────────────┐
        │                        │                  │
        ▼                        ▼                  ▼
    【入力】            【候補選択要求】      【その他】
  appendToMarked       enterCandidate      stopComposition
  insertAtCursor       SelectionMode       commitMarkedText
        │                   │                    │
        │                   ▼                    │
        │          ┌───────────────────┐        │
        │          │ SegmentsManager   │        │
        │          │ .update()         │        │
        │          └─────┬─────────────┘        │
        │                │                      │
        │                ▼                      │
        │      updateRawCandidate()             │
        │                │                      │
        │                ▼                      │
        │     KanaKanjiConverter                │
        │     .requestCandidates()              │
        │                │                      │
        │                ▼                      │
        │     ConversionResult                  │
        │     .mainResults                      │
        │     .firstClauseResults               │
        │                │                      │
        │                ▼                      │
        │     SegmentsManager.candidates        │
        │     (フィルタリング)                   │
        │                │                      │
        ▼                ▼                      ▼
    ┌──────────────────────────────────────────┐
    │     SegmentsManager 状態管理              │
    │     - shouldShowCandidateWindow           │
    │     - rawCandidates                       │
    │     - composingText                       │
    └────────────┬─────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────┐
    │  SegmentsManager.getCurrentCandidateWindow()
    │  inputState に応じて CandidateWindow を決定
    └────────────┬─────────────────────────────┘
                 │
        ┌────────▼──────────┐
        │  CandidateWindow   │
        │  enum             │
        └────────┬──────────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
    .hidden         .selecting([Candidate])
                    .composing([Candidate])
```

### 1.2 データフロー

**主要な処理フロー**:
```
ユーザ入力 → handleClientAction() → update() →
updateRawCandidate() → requestCandidates() →
ConversionResult → candidates プロパティ →
getCurrentCandidateWindow() → UI 表示
```

---

## 2. 主要メソッドの詳細分析

### 2.1 updateRawCandidate() の動作フロー

**ファイル**: `SegmentsManager.swift`
**行番号**: 390-414

**処理ステップ**:

```
【SegmentsManager.updateRawCandidate()】
│
├─→ 1. 入力テキスト確認
│   composingText.isEmpty? ──YES──→ rawCandidates = nil, 終了
│                           │
│                          NO
│                           ▼
│
├─→ 2. ユーザ辞書読み込み
│   Config.UserDictionary().items
│   └─→ 各単語のカナ → カタカナ変換
│   └─→ CID: 固有名詞(0x0000), MID: 一般(0x0000), value: -5
│   └─→ kanaKanjiConverter.importDynamicUserDictionary()
│
├─→ 3. テキスト取得
│   composingText.prefixToCursorPosition()
│   ↓
│   getCleanLeftSideContext(maxCount: 30)  [IMK 経由で取得]
│
├─→ 4. オプション生成
│   options(leftSideContext:, requestRichCandidates:)
│   ├─ requireJapanesePrediction: false ← [予測無効]
│   ├─ requireEnglishPrediction: false  ← [予測無効]
│   ├─ keyboardLanguage: .ja_JP
│   ├─ learningType: ユーザ設定
│   ├─ zenzaiMode(): Zenzai v3 設定
│   ├─ metadata: バージョン情報
│   └─ textReplacer, specialCandidateProviders: デフォルト
│
├─→ 5. 変換エンジン呼び出し
│   kanaKanjiConverter.requestCandidates(
│       prefixComposingText,
│       options: ConvertRequestOptions
│   )
│
├─→ 6. 結果保存
│   self.rawCandidates = ConversionResult
│   └─ mainResults: [Candidate]
│   └─ firstClauseResults: [Candidate]
│
└─→ 完了
```

**重要な特徴**:
- `@MainActor` で実行（スレッドセーフティ）
- 同期処理（非同期ではない）
- 毎回ユーザ辞書をインポート（パフォーマンス考慮の余地）

**コード例**:
```swift
@MainActor private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    // 入力確認
    if composingText.isEmpty {
        self.rawCandidates = nil
        self.kanaKanjiConverter.stopComposition()
        return
    }

    // ユーザ辞書統合
    var userDictionary: [DicdataElement] = userDictionary.items.map { ... }
    self.kanaKanjiConverter.importDynamicUserDictionary(userDictionary)

    // テキスト取得
    let prefixComposingText = self.composingText.prefixToCursorPosition()
    let leftSideContext = forcedLeftSideContext ??
        self.getCleanLeftSideContext(maxCount: 30)

    // 変換実行
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates
        )
    )

    // 結果保存
    self.rawCandidates = result
}
```

### 2.2 candidates プロパティ（フィルタリングロジック）

**ファイル**: `SegmentsManager.swift`
**行番号**: 338-357

**処理フロー**:

```
【SegmentsManager.candidates】
│
├─ rawCandidates が存在?
│  │
│  NO → nil を返す
│  │
│  YES
│       ▼
│    didExperienceSegmentEdition == false?
│    (テキスト分割編集の有無)
│    │
│    YES: 未編集
│    │       ▼
│    │    firstClauseResults に全文マッチする候補が存在?
│    │    │
│    │    YES → mainResults のみ返す
│    │    │
│    │    NO  → firstClauseResults + mainResults を統合
│    │          （重複除外）
│    │
│    NO: 編集済み
│        ▼
│        mainResults のみ返す
│
└─ 確定候補はこれを UI に表示
```

**コード例**:
```swift
private var candidates: [Candidate]? {
    if let rawCandidates {
        if !self.didExperienceSegmentEdition {
            if rawCandidates.firstClauseResults.contains(
                where: {
                    self.composingText.isWholeComposingText(
                        composingCount: $0.composingCount
                    )
                }
            ) {
                return rawCandidates.mainResults
            } else {
                let seenAsFirstClauseResults =
                    rawCandidates.firstClauseResults.mapSet(transform: \.text)
                return rawCandidates.firstClauseResults +
                    rawCandidates.mainResults.filter {
                        !seenAsFirstClauseResults.contains($0.text)
                    }
            }
        } else {
            return rawCandidates.mainResults
        }
    } else {
        return nil
    }
}
```

### 2.3 getCurrentCandidateWindow()

**ファイル**: `SegmentsManager.swift`
**行番号**: 482-505

**ロジック**:
```
switch inputState:
  case .none, .previewing, .replaceSuggestion:
    → .hidden

  case .composing:
    liveConversionEnabled を確認
      ↓ false: mainResults.first だけを表示 (.composing)
      ↓ true: 隠す (.hidden)

  case .selecting:
    デバッグモード確認
      ↓ true: debugCandidates を表示
      ↓ false:
         shouldShowCandidateWindow が true かつ candidates が存在?
           ↓ YES: 候補リスト表示
           ↓ NO: 隠す
```

---

## 3. データ構造の詳細

### 3.1 ConversionResult の構造（推測）

```swift
struct ConversionResult {
    var mainResults: [Candidate]
    // 通常の変換候補
    // スコアリング: 履歴学習 + Zenzai スコア
    // 用途: 標準的な候補表示

    var firstClauseResults: [Candidate]
    // 句の先頭用候補
    // 用途: composingText 全体が該当する場合の優先表示

    // 予測候補が存在する可能性（要確認）
    // var predictionResults: [Candidate]?
}
```

### 3.2 Candidate の構造（推測）

```swift
struct Candidate {
    var text: String
    // 表示するテキスト

    var value: Double
    // スコア値（低い方が優先）

    var composingCount: ComposingCount
    // この候補を確定したら消費する入力数

    var lastMid: UInt32
    // 形態素 ID（次の候補取得時に使用）

    var data: [DicdataElement]
    // 辞書データ本体
}
```

### 3.3 ConvertRequestOptions の構造

```swift
struct ConvertRequestOptions {
    var requireJapanesePrediction: Bool  // 現在: false
    var requireEnglishPrediction: Bool   // 現在: false
    var keyboardLanguage: KeyboardLanguage
    var englishCandidateInRoman2KanaInput: Bool
    var fullWidthRomanCandidate: Bool
    var learningType: LearningType
    var memoryDirectoryURL: URL
    var sharedContainerURL: URL
    var textReplacer: TextReplacer
    var specialCandidateProviders: [SpecialCandidateProvider]
    var zenzaiMode: ZenzaiMode
    var metadata: Metadata
}
```

---

## 4. 予測変換統合の設計分析

### 4.1 統合方式の比較

#### オプション A: ConversionResult に統合（推奨）

**実装概要**:
```swift
// options() メソッドの変更
var predictionOptions = options
predictionOptions.requireJapanesePrediction = true  // ← 変更

let result = kanaKanjiConverter.requestCandidates(
    prefixComposingText,
    options: predictionOptions
)
// result.predictionResults を利用（存在すれば）
```

**メリット**:
1. **API 統一性**: 1回の呼び出しで全候補取得
2. **スコアリング一貫性**: エンジン側で統一的に処理
3. **実装の単純性**: updateRawCandidate() の変更が最小限
4. **パフォーマンス**: API 呼び出しが 1 回で済む

**デメリット**:
1. **エンジン依存**: ConversionResult の構造に依存
2. **制御の制限**: 予測と変換を独立制御しにくい
3. **コンテキスト共有**: 同じ左文脈を使用

**変更行数**: 約 15 行

#### オプション B: 別構造で管理

**実装概要**:
```swift
// SegmentsManager に新規プロパティ
private var predictionCandidates: [Candidate]?

// 新しい更新メソッド
private func updatePredictionCandidates() async {
    let result = kanaKanjiConverter.requestPredictionCandidates(...)
    self.predictionCandidates = result
}
```

**メリット**:
1. **独立制御**: 予測と変換を完全分離
2. **別タイミング取得**: 非同期でプリフェッチ可能
3. **UI 柔軟性**: 別ウィンドウ表示など自由に設計

**デメリット**:
1. **実装複雑**: 新規メソッド、プロパティが必要
2. **API 呼び出し増**: 2回の呼び出しが必要
3. **同期の課題**: データ同期が複雑化

**変更行数**: 約 100-150 行

#### オプション C: ハイブリッド方式（最推奨）

**実装概要**:
```swift
// updateRawCandidate() で predictionResults も取得
let result = kanaKanjiConverter.requestCandidates(
    options: options(requireJapanesePrediction: true)
)

// candidates プロパティで統合
private var candidates: [Candidate]? {
    guard let rawCandidates else { return nil }

    let main = rawCandidates.mainResults
    let first = rawCandidates.firstClauseResults
    let prediction = rawCandidates.predictionResults ?? []

    // 重複除去して統合
    return integratedCandidates(main, first, prediction)
}
```

**メリット**:
- オプション A のシンプルさ
- オプション B の柔軟性
- 段階的な拡張が可能

**変更行数**: 約 15-30 行

### 4.2 推奨実装ロードマップ

#### Phase 1: 基本統合（1-2 週間）

**タスク**:
1. AzooKeyKanaKanjiConverter の API 確認
   - predictionResults の存在確認
   - requireJapanesePrediction の動作確認
2. options() で `requireJapanesePrediction = true` に変更
3. candidates プロパティに予測候補統合ロジック追加
4. 基本動作テスト

**成果物**:
- 予測候補が表示される基本機能
- ユニットテスト

**変更行数**: 約 15 行

#### Phase 2: UI 表示改善（1 週間）

**タスク**:
1. 予測候補と通常候補の視覚的区分
2. CandidatesViewController の修正
3. UI テスト

**成果物**:
- 予測候補が青色で表示される
- セパレーター表示

**変更行数**: 約 30-35 行

#### Phase 3: ユーザ設定（1 週間）

**タスク**:
1. 予測変換の有効/無効設定追加
2. 予測候補の最大表示件数設定
3. Settings UI 拡張

**成果物**:
- Config.PredictionEnabled
- Config.PredictionMaxCandidates

#### Phase 4: パフォーマンス最適化（2-3 週間）

**タスク**:
1. 非同期化の検討
2. キャッシング機構
3. メモリ使用量最適化

#### Phase 5: 高度な機能（3-4 週間）

**タスク**:
1. Zenzai 統合
2. 多段階予測
3. 言語モデル選択

**総期間**: 10-14 週間

---

## 5. 発見された問題点と制約

### 5.1 パフォーマンス関連

1. **毎回のユーザ辞書インポート**
   - 影響: updateRawCandidate() のたびに全辞書を読み込み
   - 対策: キャッシング、差分管理

2. **同期的な候補取得**
   - 影響: Zenzai 推論時間が長い場合の UI フリーズ
   - 対策: 非同期化、タイムアウト設定

### 5.2 機能的な制約

3. **予測機能が無効化**
   - 現状: requireJapanesePrediction = false
   - 対策: true に変更して動作確認

4. **編集フラグの依存性**
   - 影響: didExperienceSegmentEdition が不正な状態の可能性
   - 対策: テスト追加、フラグ管理の見直し

### 5.3 拡張性の課題

5. **ConversionResult の拡張性不明**
   - 影響: predictionResults が存在しない可能性
   - 対策: AzooKeyKanaKanjiConverter の API 確認必須

6. **左文脈の制限**
   - 現状: 最大 30 文字
   - 影響: Zenzai が必要とする文脈量と不一致の可能性

---

## 6. リスク評価

| リスク | 確率 | 影響度 | 対策 |
|------|------|--------|------|
| ConversionResult に predictionResults がない | 中 | 高 | API 確認、代替実装 |
| パフォーマンス劣化 | 中 | 中 | 非同期化、キャッシング |
| UI の複雑化 | 低 | 中 | 段階的実装、ユーザーテスト |
| 状態遷移バグ | 低 | 高 | テスト追加、ログ強化 |

---

## 7. 次のステップ

### 即座に実施（Week 1）

1. **API 確認**
   - [ ] AzooKeyKanaKanjiConverter のドキュメント確認
   - [ ] predictionResults フィールドの存在確認
   - [ ] requireJapanesePrediction の動作確認

2. **プロトタイプ A**
   - [ ] options() で requireJapanesePrediction = true に変更
   - [ ] ビルド・実行テスト
   - [ ] 予測候補が取得されるか確認

### 中期実施（Week 2-3）

3. **パフォーマンス測定**
   - [ ] updateRawCandidate() の実行時間測定
   - [ ] メモリ使用量測定
   - [ ] Zenzai 推論時間測定

4. **ユニットテスト追加**
   - [ ] candidates プロパティのテスト
   - [ ] updateRawCandidate() のテスト

### 長期実施（Week 4-5）

5. **設計ドキュメント作成**
6. **UI/UX デザイン検討**
7. **テストケース作成**

---

## 8. 結論

azooKey-Desktop の変換候補取得システムは、予測変換機能の追加に適した構造を持っています。

**実現可能性**: ⭐⭐⭐⭐⭐（非常に高い）
**実装複雑性**: ⭐⭐（低い）
**推奨アプローチ**: ハイブリッド方式（ConversionResult 統合 + 独立表示）

**次のクリティカルステップ**: AzooKeyKanaKanjiConverter の API 確認

---

## 付録: 調査データ

### 調査対象ファイル

1. `/Users/kosei.yamamoto/git/azookey/azooKey-Desktop/azooKeyMac/InputController/SegmentsManager.swift` (675 行)
2. `/Users/kosei.yamamoto/git/azookey/azooKey-Desktop/azooKeyMac/InputController/azooKeyMacInputController.swift` (648 行)
3. 外部依存: KanaKanjiConverterModuleWithDefaultDictionary

### 主要メソッド一覧

| メソッド | 行番号 | 説明 |
|---------|--------|------|
| updateRawCandidate() | 390-414 | 候補取得のメイン処理 |
| options() | 182-197 | ConvertRequestOptions 生成 |
| zenzaiMode() | 153-170 | Zenzai 設定 |
| candidates | 338-357 | 候補フィルタリング |
| getCurrentCandidateWindow() | 482-505 | UI 表示制御 |
| update() | 416-419 | 外部インターフェース |

### 参考資料

- 実装計画書: `.claude/prediction-feature-implementation-plan.md`
- プロジェクト知見: `.claude/project-knowledge.md`
- コンテキスト: `.claude/context.md`
- Serena メモリ: `conversion_system_architecture`, `prediction_integration_analysis`, `detailed_investigation_report`
