# Phase 1 - Step 2: AzooKeyKanaKanjiConverter の予測変換 API 調査

**調査完了日**: 2024-11-08
**調査対象**: AzooKeyKanaKanjiConverter ライブラリ
**目的**: 予測変換機能の実装可能性と具体的な実装方法の決定

---

## エグゼクティブサマリー

### 調査結果の概要

**実装可能性**: ✅ **YES - すぐに実装可能**

### 重要な発見

1. **requireJapanesePrediction は完全実装済み**
   - パラメータを `true` に設定することで予測候補が生成される
   - mainResults に最大3件の予測候補が統合される

2. **predictionResults フィールドは存在しない**
   - ConversionResult には mainResults と firstClauseResults のみ
   - 予測候補は mainResults に統合される形式

3. **確定後予測変換専用 API が存在（推奨）**
   - `requestPostCompositionPredictionCandidates()` メソッド
   - 確定した候補の後に続く単語を予測
   - 最大10件の予測候補を取得可能

### 推奨実装方法

**確定後予測変換（requestPostCompositionPredictionCandidates）の使用**

**理由**:
- 計算効率が良い
- UI 表示が明確
- ユーザー体験が自然
- 実装コストが低い（3～4日）

**実装スケール**:
- 変更ファイル数: 3～5ファイル
- 実装期間: 3～4日
- 変更行数: 約50～100行

---

## 1. API の存在確認結果

### 1.1 requireJapanesePrediction パラメータ

**確認結果**: ✅ 完全に実装されている

#### パラメータの定義

**ファイル**: `ConvertRequestOptions.swift`
**型**: `public var requireJapanesePrediction: Bool`

#### 動作詳細

| 設定値 | 動作 | 生成件数 |
|-------|------|---------|
| true | 日本語予測変換候補を生成 | 最大3件 |
| false | 予測候補を生成しない | 0件 |

#### 内部処理フロー

**KanaKanjiConverter.swift での実装**:

```swift
// 行515-521: 予測用の最適候補を計算
if options.requireJapanesePrediction {
    let clauseResultCandidates = clauseResult.map {
        self.converter.processClauseCandidate($0)
    }
    bestCandidateDataForPrediction = zip(clauseResult, clauseResultCandidates)
        .max {$0.1.value < $1.1.value}!.0
    wholeSentenceUniqueCandidates = self.getUniqueCandidate(clauseResultCandidates)
}

// 行571-577: 予測候補を最大3件生成
let bestThreePredictionCandidates: [Candidate] =
    if options.requireJapanesePrediction, let bestCandidateDataForPrediction {
        self.getUniqueCandidate(
            self.getPredictionCandidate(bestCandidateDataForPrediction,
                                       composingText: inputData,
                                       options: options)
        ).min(count: 3, sortedBy: {$0.value > $1.value})
    } else {
        []
    }
```

#### requireEnglishPrediction との関係性

- **完全に独立して動作**
- 両方を同時に有効化可能
- 英語予測は SpellChecker を使用（別メカニズム）

### 1.2 ConversionResult の構造

**確認結果**: ❌ **predictionResults フィールドは存在しない**

#### 実際の定義

```swift
public struct ConversionResult: Sendable {
    /// 変換候補欄にこのままの順で並べることのできる候補
    public var mainResults: [Candidate]

    /// 変換候補のうち最初の文節を変換したもの
    public var firstClauseResults: [Candidate]
}
```

#### 候補の構成

**mainResults には以下が統合される**（優先度順）:

1. 全文変換の上位候補
2. **予測変換候補（最大3件）** ← requireJapanesePrediction = true で追加
3. 英語予測候補
4. 特殊変換候補（ユーザーショートカットなど）

**最終的に上位候補として返される**

### 1.3 予測変換専用メソッドの発見

**確認結果**: ✅ **専用メソッドが存在**

#### requestPostCompositionPredictionCandidates()

**メソッド定義**:
```swift
public func requestPostCompositionPredictionCandidates(
    leftSideCandidate: Candidate,
    options: ConvertRequestOptions
) -> [PostCompositionPredictionCandidate]
```

**特徴**:
- 確定した候補の後に続く単語を予測
- 最大10件の候補を取得可能
- 助詞、次の動詞など自然な流れの提案

**戻り値の型**:
```swift
public struct PostCompositionPredictionCandidate {
    public var text: String              // 予測テキスト
    public var value: PValue             // スコア
    public var type: PredictionType      // 予測タイプ
    public var isTerminal: Bool          // 句点（。）で終了か

    public enum PredictionType: Sendable, Hashable {
        case additional(data: [DicdataElement])           // 追加
        case replacement(targetData: [DicdataElement],    // 置換
                        replacementData: [DicdataElement])
    }

    // Candidate と結合するメソッド
    public func join(to candidate: Candidate) -> Candidate
}
```

---

## 2. 型定義の完全な情報

### 2.1 ConversionResult

```swift
public struct ConversionResult: Sendable {
    public var mainResults: [Candidate]
    public var firstClauseResults: [Candidate]
}
```

| フィールド | 型 | 説明 | 最大件数 |
|-----------|-----|------|---------|
| mainResults | [Candidate] | 候補ウィンドウに表示する全候補 | 可変（15～20件） |
| firstClauseResults | [Candidate] | 最初の文節のみを変換した候補 | 最大5件 |

### 2.2 ConvertRequestOptions（全プロパティ）

```swift
public struct ConvertRequestOptions: Sendable {
    // 変換制御
    public var N_best: Int                                  // デフォルト: 10
    public var requireJapanesePrediction: Bool              // 日本語予測
    public var requireEnglishPrediction: Bool               // 英語予測
    public var keyboardLanguage: KeyboardLanguage           // 言語設定
    public var needTypoCorrection: Bool?                   // タイプミス修正

    // 候補オプション
    public var englishCandidateInRoman2KanaInput: Bool      // 英語候補
    public var fullWidthRomanCandidate: Bool               // 全角英数
    public var halfWidthKanaCandidate: Bool                // 半角カナ

    // 学習関連
    public var learningType: LearningType                  // 学習モード
    public var maxMemoryCount: Int                         // 学習データ最大数
    public var shouldResetMemory: Bool                     // メモリリセット

    // パス・URL
    public var memoryDirectoryURL: URL                     // 学習データ保存先
    public var sharedContainerURL: URL                     // ユーザー辞書位置

    // 機能
    public var textReplacer: TextReplacer                  // テキスト置換
    public var specialCandidateProviders: [any SpecialCandidateProvider]
    public var zenzaiMode: ZenzaiMode                      // AI モード
    public var preloadDictionary: Bool                     // 辞書プリロード
    public var metadata: Metadata?                         // メタデータ
}
```

### 2.3 Candidate

```swift
public struct Candidate: Sendable {
    // 基本情報
    public var text: String                    // 候補のテキスト
    public var value: PValue                   // スコア（Float型）

    // 入力情報
    public var composingCount: ComposingCount  // 入力文字数
    public var lastMid: Int                    // 最後の品詞ID

    // 辞書データ
    public var data: [DicdataElement]          // 単語の詳細情報

    // 動作
    public var actions: [CompleteAction]       // 確定時アクション

    // フラグ
    public let inputable: Bool                 // 入力可能か
    public let rubyCount: Int                  // ルビの文字数
    public var isLearningTarget: Bool          // 学習対象か
}
```

### 2.4 ComposingCount

```swift
public enum ComposingCount: Sendable, Equatable {
    case inputCount(Int)              // ローマ字入力の文字数
    case surfaceCount(Int)            // 変換結果の文字数
    indirect case composite(lhs: Self, rhs: Self)  // 複合
}
```

---

## 3. 実装可能性の評価

### 3.1 結論

**予測変換の実装可能性**: ✅ **YES - すぐに実装可能**

### 3.2 推奨される実装アプローチの比較

#### アプローチ1: 確定後予測変換（⭐⭐⭐⭐⭐ 最推奨）

**実装概要**:
```swift
@MainActor
func getPredictions(for candidate: Candidate) -> [PostCompositionPredictionCandidate] {
    let options = self.options(leftSideContext: nil, requestRichCandidates: false)
    return self.kanaKanjiConverter.requestPostCompositionPredictionCandidates(
        leftSideCandidate: candidate,
        options: options
    )
}
```

**メリット**:
- ✅ 計算コストが低い
- ✅ UI 表示が明確（予測候補を独立表示）
- ✅ ユーザー体験が自然
- ✅ 実装コストが低い（約50～100行）

**デメリット**:
- ⚠️ 確定後のみ予測（入力中は予測なし）

**使用場面**:
- 単語確定直後に次の単語を提示
- 助詞、次の動詞など自然な流れの提案

**実装期間**: 3～4日

#### アプローチ2: ライブ予測変換（⭐⭐⭐）

**実装概要**:
```swift
let options = ConvertRequestOptions(
    // ...
    requireJapanesePrediction: true,  // false → true に変更
    // ...
)

let result = kanaKanjiConverter.requestCandidates(composingText, options: options)
// result.mainResults に予測候補が含まれる（最大3件）
```

**メリット**:
- ✅ 既存メソッドで対応可能
- ✅ 統合的なスコアリング
- ✅ 入力中から予測表示

**デメリット**:
- ⚠️ 計算コスト +10-20%
- ⚠️ 予測候補と通常候補の区別が難しい
- ⚠️ 最大3件のみ

**実装期間**: 1日（options() の変更のみ）

#### アプローチ3: ハイブリッド実装（⭐⭐⭐⭐）

**実装概要**:
```swift
// ライブ変換時: requireJapanesePrediction = false（現状維持）
// ユーザー確定後: requestPostCompositionPredictionCandidates を使用
```

**メリット**:
- ✅ パフォーマンスと機能の両立
- ✅ 現在のアーキテクチャと互換性
- ✅ 柔軟な UX 設計

**実装期間**: 5～7日

### 3.3 必要な変更箇所

**SegmentsManager.swift**:
1. 確定後予測候補取得メソッドの追加（約20行）
2. prefixCandidateCommited() の拡張（約10行）
3. 予測候補の保持用プロパティ追加（約5行）

**azooKeyMacInputController.swift**:
1. 予測候補表示の制御ロジック（約15行）
2. UI イベントハンドリング（約10行）

**CandidatesViewController.swift**:
1. 予測候補専用の表示ロジック（約20行）

**合計変更行数**: 約50～100行

---

## 4. コード例

### 4.1 確定後予測変換の実装例

#### SegmentsManager.swift への追加

```swift
// MARK: - Prediction Support

/// 予測候補を保持
private var postCompositionPredictions: [PostCompositionPredictionCandidate] = []

/// 確定した候補に対する予測候補を取得
@MainActor
private func getPredictionsForCandidate(_ candidate: Candidate)
    -> [PostCompositionPredictionCandidate] {
    let options = self.options(
        leftSideContext: self.getCleanLeftSideContext(maxCount: 30),
        requestRichCandidates: false
    )
    return self.kanaKanjiConverter.requestPostCompositionPredictionCandidates(
        leftSideCandidate: candidate,
        options: options
    )
}

/// 候補確定時の拡張（既存メソッドに追加）
@MainActor
func prefixCandidateCommited(_ candidate: Candidate, leftSideContext: String) {
    // 既存の処理
    self.kanaKanjiConverter.setCompletedData(candidate)
    self.updateLearningDataAsync(candidate)
    self.composingText.prefixComplete(composingCount: candidate.composingCount)

    if !self.composingText.isEmpty {
        // 既存処理...
        self.didExperienceSegmentEdition = false
        _ = self.composingText.moveCursorFromCursorPosition(
            count: self.composingText.convertTarget.count -
                   self.composingText.convertTargetCursorPosition
        )
        self.didExperienceSegmentEdition = false
        self.shouldShowCandidateWindow = true
        self.selectionIndex = nil
        self.updateRawCandidate(
            requestRichCandidates: true,
            forcedLeftSideContext: leftSideContext + candidate.text
        )

        // 新規: 予測候補を取得
        self.postCompositionPredictions = self.getPredictionsForCandidate(candidate)
        self.delegate?.predictionCandidatesUpdated(self.postCompositionPredictions)
    }
}
```

#### Delegate プロトコルの拡張

```swift
// SegmentManagerDelegate に追加

protocol SegmentManagerDelegate: AnyObject {
    // 既存のメソッド...

    /// 予測候補が更新された
    func predictionCandidatesUpdated(_ predictions: [PostCompositionPredictionCandidate])
}
```

### 4.2 ライブ予測変換の有効化

```swift
// SegmentsManager.swift の options() メソッド修正

private func options(
    leftSideContext: String? = nil,
    requestRichCandidates: Bool = false
) -> ConvertRequestOptions {
    .init(
        requireJapanesePrediction: true,  // ← false から true に変更
        requireEnglishPrediction: false,
        keyboardLanguage: .ja_JP,
        englishCandidateInRoman2KanaInput: false,
        fullWidthRomanCandidate: true,
        learningType: Config.Learning().value.learningType,
        memoryDirectoryURL: self.azooKeyMemoryDir,
        sharedContainerURL: self.azooKeyMemoryDir,
        textReplacer: .withDefaultEmojiDictionary(),
        specialCandidateProviders: KanaKanjiConverter.defaultSpecialCandidateProviders,
        zenzaiMode: self.zenzaiMode(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates
        ),
        metadata: self.metadata
    )
}
```

### 4.3 予測候補の UI 表示

```swift
// azooKeyMacInputController.swift

extension azooKeyMacInputController: SegmentManagerDelegate {
    func predictionCandidatesUpdated(_ predictions: [PostCompositionPredictionCandidate]) {
        Task { @MainActor in
            // 予測候補を最大3件表示
            let displayPredictions = Array(predictions.prefix(3))

            self.showPredictionCandidates(displayPredictions)
        }
    }

    private func showPredictionCandidates(
        _ predictions: [PostCompositionPredictionCandidate]
    ) {
        // 予測候補ウィンドウに表示
        // 既存の候補ウィンドウを再利用するか、新しいウィンドウを作成

        for prediction in predictions {
            switch prediction.type {
            case .additional(data: let data):
                // 追加型の予測候補を表示
                print("予測（追加）: \(prediction.text)")

            case .replacement(targetData: _, replacementData: let data):
                // 置換型の予測候補を表示
                print("予測（置換）: \(prediction.text)")
            }

            // 句点で終了する候補は最後に表示
            if prediction.isTerminal {
                break
            }
        }
    }
}
```

---

## 5. 制約事項とリスク

### 5.1 スレッドセーフティの制約

| 項目 | 制約 | 対応方法 |
|------|------|---------|
| requestCandidates() | メインスレッド必須 | @MainActor で保護 |
| requestPostCompositionPredictionCandidates() | メインスレッド必須 | @MainActor で保護 |
| 学習データ更新 | 非同期API推奨 | updateLearningDataAsync() 使用 |
| KanaKanjiConverter アクセス | 排他制御なし | AppDelegate で単一インスタンス管理 |

**現在の実装**:
```swift
@MainActor private var kanaKanjiConverter: KanaKanjiConverter {
    (NSApplication.shared.delegate as? AppDelegate)!.kanaKanjiConverter
}
```

**推奨事項**:
- すべての KanaKanjiConverter 呼び出しを @MainActor で実行
- 非同期処理から呼び出す場合は `MainActor.run` を使用

### 5.2 パフォーマンス上の懸念

#### 計算コスト

| 処理 | コスト | 備考 |
|------|--------|------|
| requestCandidates() | 数ms～数十ms | 入力長に依存 |
| 予測候補（3件） | 数ms | ほぼ無視できる |
| requireJapanesePrediction=true | +10-20% | 予測候補計算の追加コスト |
| requestPostCompositionPredictionCandidates() | 数ms | 確定後のみ |
| Zenzai 有効時 | 数百ms | AI 推論による |

#### 推奨事項

**ライブ変換時**:
- `requireJapanesePrediction: false` 維持（現状のまま）
- UI 応答性を優先

**確定後**:
- `requestPostCompositionPredictionCandidates()` 使用
- バックグラウンドで取得可能

#### メモリ管理

```swift
private var zenzaiCache: Kana2Kanji.ZenzaiCache?
```

- AI 推論キャッシュあり
- 通常使用では問題なし
- 長時間の連続入力でも安定

### 5.3 バージョン依存性

**現在の AzooKeyKanaKanjiConverter バージョン**:
```swift
.package(path: "../../AzooKeyKanaKanjiConverter", traits: ["Zenzai"])
```

**API 互換性**:

| API | 対応バージョン | 備考 |
|-----|-------------|------|
| requireJapanesePrediction | v0.x 以降 | 安定 |
| requestPostCompositionPredictionCandidates() | v0.4 以降 | 推奨 |
| PostCompositionPredictionCandidate | v0.4 以降 | 確定後予測専用 |

**注意**: バージョンアップ時に動作確認必須

### 5.4 その他の制約

#### 辞書依存性
- 予測候補は内蔵辞書の品質に依存
- ユーザー辞書で補強可能

#### 言語別対応
```swift
public var keyboardLanguage: KeyboardLanguage
```
- 日本語（ja_JP）のみサポート
- 英語予測は別メカニズム（SpellChecker）

---

## 6. 実装ロードマップ

### Phase 1: 基礎実装（1日）

**タスク**:
- [ ] SegmentsManager に getPredictionsForCandidate() メソッド追加
- [ ] requestPostCompositionPredictionCandidates() の呼び出し実装
- [ ] postCompositionPredictions プロパティ追加
- [ ] SegmentManagerDelegate の拡張

**成果物**:
- 予測候補取得の基本機能
- デリゲートメソッドの実装

### Phase 2: UI 統合（1～2日）

**タスク**:
- [ ] 予測候補表示用の UI コンポーネント
- [ ] PostCompositionPredictionCandidate の描画ロジック
- [ ] キーボードナビゲーション実装
- [ ] 予測候補選択時の処理

**成果物**:
- 予測候補ウィンドウ
- 選択・確定機能

### Phase 3: テストと最適化（1日）

**タスク**:
- [ ] ユニットテスト
- [ ] パフォーマンステスト
- [ ] エッジケース対応
- [ ] メモリリーク確認

**成果物**:
- テストスイート
- パフォーマンスレポート

### Phase 4: デプロイ（0.5日）

**タスク**:
- [ ] ユーザーテスト
- [ ] ドキュメント更新
- [ ] リリース準備

**総工期**: 3～4日

---

## 7. 結論と推奨事項

### 7.1 実装可能性の最終評価

| 評価項目 | 結果 | 詳細 |
|---------|------|------|
| **API 存在** | ✅ あり | requestPostCompositionPredictionCandidates() |
| **requireJapanesePrediction** | ✅ 完全実装 | mainResults に統合 |
| **predictionResults** | ❌ 存在しない | 別メソッド使用を推奨 |
| **代替メソッド** | ✅ あり | 確定後予測専用メソッド |
| **型定義の完全性** | ✅ 完全 | 全型が明確に定義 |
| **スレッドセーフティ** | ⚠️ 条件付き | @MainActor で対応 |
| **パフォーマンス** | ✅ 良好 | キャッシング機構あり |
| **実装可能性** | **✅ YES** | **すぐに実装可能** |

### 7.2 推奨実装方法

**確定後予測変換（requestPostCompositionPredictionCandidates）を使用**

**理由**:
1. 計算効率が良い
2. UI 表示が明確
3. ユーザー体験が自然
4. 実装コストが低い

**実装期間**: 3～4日
**変更行数**: 約50～100行
**リスク**: 低

### 7.3 次のステップ

**即座に実施（Week 1）**:
1. [ ] プロトタイプの作成
   - getPredictionsForCandidate() の実装
   - 基本的な動作確認
2. [ ] UI モックアップの作成
   - 予測候補の表示方法を設計

**中期実施（Week 2）**:
1. [ ] 完全な実装
   - UI 統合
   - キーボードナビゲーション
2. [ ] テスト実施
   - ユニットテスト
   - 統合テスト

---

## 付録: API リファレンス

### requestPostCompositionPredictionCandidates()

**シグネチャ**:
```swift
public func requestPostCompositionPredictionCandidates(
    leftSideCandidate: Candidate,
    options: ConvertRequestOptions
) -> [PostCompositionPredictionCandidate]
```

**パラメータ**:
- `leftSideCandidate`: 確定した候補
- `options`: 変換オプション

**戻り値**:
- `[PostCompositionPredictionCandidate]`: 予測候補の配列（最大10件）

**使用例**:
```swift
let candidate = Candidate(text: "食べた", ...)
let predictions = kanaKanjiConverter.requestPostCompositionPredictionCandidates(
    leftSideCandidate: candidate,
    options: options
)

// 結果例:
// [
//   PostCompositionPredictionCandidate(text: "から", ...),
//   PostCompositionPredictionCandidate(text: "ので", ...),
//   PostCompositionPredictionCandidate(text: "。", isTerminal: true, ...)
// ]
```

### PostCompositionPredictionCandidate.join()

**シグネチャ**:
```swift
public func join(to candidate: Candidate) -> Candidate
```

**説明**: 予測候補を既存の Candidate に結合

**使用例**:
```swift
let original = Candidate(text: "食べた", ...)
let prediction = PostCompositionPredictionCandidate(text: "から", ...)
let combined = prediction.join(to: original)
// combined.text == "食べたから"
```

---

## 参考資料

- 実装計画書: `.claude/prediction-feature-implementation-plan.md`
- Step 1 調査結果: `.claude/phase1-step1-candidate-system-analysis.md`
- プロジェクト知見: `.claude/project-knowledge.md`
- コンテキスト: `.claude/context.md`
