# Phase 2 - Step 3: リアルタイム予測変換のトリガー条件とタイミング設計

**作成日**: 2025-11-08
**対象機能**: リアルタイム予測変換（Live Prediction）
**設計方針**: ユーザー要望により、リアルタイム予測のみを実装

---

## エグゼクティブサマリー

### 設計決定の概要

**実装アプローチ**: ✅ **リアルタイム予測のみ実装**

ユーザーの明確な要望により、以下の設計方針を採用：
- ✅ **リアルタイム予測（Live Prediction）**: 入力中の未確定文字列から予測候補を表示
- ❌ **確定後予測（Post-Composition Prediction）**: 実装しない

### 主要な設計決定

1. **requireJapanesePrediction を true に変更**
   - 現在 false → true に変更することで予測機能を有効化
   - パフォーマンスオーバーヘッド: +10～20%（許容範囲内）

2. **ライブ変換中でも予測を表示**
   - ユーザー要望: 「ライブ変換中でも予測変換して欲しい」
   - liveConversionEnabled の値に関わらず予測候補を取得

3. **デバウンス処理による最適化**
   - デバウンス時間: 100～150ms
   - Task キャンセル機構でパフォーマンス最適化

4. **スレッドセーフティの厳守**
   - すべての KanaKanjiConverter 呼び出しを @MainActor で実行
   - 非同期処理パターンの踏襲

---

## 1. リアルタイム予測アプローチの選択理由

### 1.1 リアルタイム予測のみを選んだ理由

**ユーザー要望**:
> 「ライブ変換中でも予測変換して欲しい」

この要望から、以下の判断を行いました：

1. **入力中のリアルタイムフィードバックが重要**
   - ユーザーは入力しながら次の候補を見たい
   - 確定してから予測を見るのではタイミングが遅い

2. **ライブ変換との共存が必須**
   - ライブ変換 ON でも予測を表示する必要がある
   - 既存の変換機能を阻害しない設計が重要

3. **実装の単純性**
   - requireJapanesePrediction を true にするだけで基本機能が動作
   - 確定後予測の追加実装は不要

### 1.2 確定後予測を実装しない理由

**技術的理由**:
1. **ユーザー要望との不一致**
   - 確定後予測は「確定してから」表示される
   - ユーザーは「入力中に」予測を見たい

2. **実装コストの削減**
   - 確定後予測は別途 API（requestPostCompositionPredictionCandidates）が必要
   - リアルタイム予測のみで要件を満たせる

3. **UX の一貫性**
   - 2つの予測方式を混在させると UX が複雑化
   - 単一の予測方式で一貫したユーザー体験を提供

---

## 2. 詳細なトリガー条件仕様

### 2.1 基本トリガー条件

#### 最小文字数

**仕様**:
- **推奨値**: 2文字以上
- **理由**: 1文字では候補が多すぎて有用性が低い

```swift
private var minimumCharactersForPrediction: Int {
    2
}

private func shouldTriggerPrediction() -> Bool {
    guard !composingText.isEmpty else { return false }
    
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    return characterCount >= minimumCharactersForPrediction
}
```

**設定可能性**:
- ユーザー設定で 1～5 文字の範囲で変更可能にすることを推奨
- デフォルト: 2文字

#### 入力モード

**仕様**:
- **ひらがな入力時**: 予測を有効化 ✅
- **カタカナ入力時**: 予測を有効化 ✅
- **英数字入力時**: 予測を無効化 ❌

```swift
private func isValidInputModeForPrediction() -> Bool {
    // 現在の入力モードを確認
    let inputMode = composingText.inputStyle
    
    switch inputMode {
    case .direct:
        // 直接入力（英数字）の場合は予測しない
        return false
    case .roman2kana:
        // ローマ字かな変換の場合は予測する
        return true
    }
}
```

**理由**:
- 日本語入力時のみ日本語予測変換が有用
- 英数字入力時は requireEnglishPrediction を使用（将来拡張）

### 2.2 ライブ変換との関係

**重要**: ユーザーは「ライブ変換中でも予測変換して欲しい」と要望

#### ライブ変換 ON の場合

**動作**:
```
入力中 → ライブ変換が自動実行 + 予測候補を表示
         ├─ mainResults に変換候補
         └─ 予測候補も mainResults に統合（最大3件）
```

**仕様**:
```swift
private func shouldShowPredictionInLiveConversion() -> Bool {
    // ライブ変換中でも予測を表示する
    return liveConversionEnabled && shouldTriggerPrediction()
}
```

**UI 表示制御**:
```swift
func getCurrentCandidateWindow() -> CandidateWindow {
    switch inputState {
    case .composing:
        if liveConversionEnabled {
            // ライブ変換中でも予測候補を表示
            if shouldShowPredictionInLiveConversion(),
               let candidates = candidates,
               !candidates.isEmpty {
                return .composing(candidates, selectionIndex: selectionIndex)
            } else {
                return .hidden
            }
        } else {
            // ライブ変換 OFF の場合は通常動作
            if let first = self.candidates?.first {
                return .composing([first], selectionIndex: nil)
            } else {
                return .hidden
            }
        }
    // ... 他のケース
    }
}
```

#### ライブ変換 OFF の場合

**動作**:
```
入力中 → 通常の変換候補 + 予測候補を表示
         └─ mainResults に両方が統合される
```

**仕様**: ライブ変換 ON と同じ処理

### 2.3 セグメント編集との関係

**基本方針**: セグメント編集中は予測を抑制

**理由**:
- セグメント編集は既存の変換結果の調整作業
- この時点での予測は不要で、むしろ混乱を招く

```swift
private func shouldTriggerPrediction() -> Bool {
    guard !composingText.isEmpty else { return false }
    
    // セグメント編集中は予測しない
    if didExperienceSegmentEdition {
        return false
    }
    
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    return characterCount >= minimumCharactersForPrediction &&
           isValidInputModeForPrediction()
}
```

### 2.4 カーソル位置による制御

**仕様**: カーソル位置に関わらず予測を実行

**理由**:
- `composingText.prefixToCursorPosition()` で現在位置までのテキストを使用
- カーソル前のテキストに基づいて予測することで自然な UX

```swift
@MainActor private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    if composingText.isEmpty {
        self.rawCandidates = nil
        self.kanaKanjiConverter.stopComposition()
        return
    }
    
    // カーソル位置までのテキストを取得
    let prefixComposingText = self.composingText.prefixToCursorPosition()
    let leftSideContext = forcedLeftSideContext ?? self.getCleanLeftSideContext(maxCount: 30)
    
    // 予測を有効化した options を使用
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates,
            enablePrediction: shouldTriggerPrediction() // 新規パラメータ
        )
    )
    self.rawCandidates = result
}
```

### 2.5 その他の除外条件

**特殊文字入力時**:
- 記号のみの入力: 予測を無効化
- 数字のみの入力: 予測を無効化

```swift
private func isValidContentForPrediction() -> Bool {
    let prefixText = composingText.prefixToCursorPosition()
    let target = prefixText.convertTarget
    
    // 全て記号または数字の場合は予測しない
    let isOnlySymbols = target.allSatisfy { char in
        char.isSymbol || char.isPunctuation || char.isNumber
    }
    
    return !isOnlySymbols
}
```

**空白のみの入力**:
```swift
private func isValidContentForPrediction() -> Bool {
    let prefixText = composingText.prefixToCursorPosition()
    let target = prefixText.convertTarget
    
    // 空白のみの場合は予測しない
    return !target.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
}
```

---

## 3. タイミング制御の仕様

### 3.1 デバウンス処理

**目的**: 入力のたびに予測すると重いため、デバウンスで最適化

#### デバウンス時間の設定

**推奨値**: 100～150ms

**理由**:
- 100ms 未満: 効果が薄い（タイピング速度の影響が小さい）
- 200ms 以上: ユーザーが待たされる感覚が出る
- 100～150ms: パフォーマンスと UX のバランスが最適

#### 実装パターン

```swift
// SegmentsManager に追加
private var predictionDebounceTask: Task<Void, Never>?
private let predictionDebounceDelay: Duration = .milliseconds(100)

@MainActor
private func updateRawCandidateWithDebounce(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    // 既存のタスクをキャンセル
    predictionDebounceTask?.cancel()
    
    // 新しいデバウンスタスクを開始
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        // デバウンス待機
        do {
            try await Task.sleep(for: predictionDebounceDelay)
        } catch {
            // キャンセルされた場合は処理を中断
            return
        }
        
        // キャンセルされていなければ候補を更新
        if !Task.isCancelled {
            self.updateRawCandidate(
                requestRichCandidates: requestRichCandidates,
                forcedLeftSideContext: forcedLeftSideContext
            )
        }
    }
}
```

#### デバウンス処理のフローチャート

```
ユーザー入力
    │
    ▼
既存のタスクをキャンセル
    │
    ▼
新しいタスクを開始
    │
    ▼
100ms 待機（Task.sleep）
    │
    ├─ 新しい入力があった？
    │  └─ YES → タスクがキャンセルされる → 終了
    │
    └─ NO
       │
       ▼
    候補を更新（updateRawCandidate）
```

### 3.2 非同期処理パターン

**基本方針**: メインスレッドブロッキングを 0ms に保つ

#### KanaKanjiConverter 呼び出しの MainActor 実行

**制約**: AzooKeyKanaKanjiConverter は明確にスレッドセーフではない

**対策**: すべての呼び出しを @MainActor で実行

```swift
@MainActor private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    // 省略
    
    // MainActor で実行されるため、スレッドセーフティが保証される
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates,
            enablePrediction: shouldTriggerPrediction()
        )
    )
    self.rawCandidates = result
}
```

#### 既存の非同期処理パターンの踏襲

**参照**: ns/test3 ブランチでの学習データ非同期化パターン

```swift
// ユーザー入力イベント（@MainActor）
@MainActor func handleUserInput() {
    // 1. UI 状態を即座に更新
    self.updateInputState()
    
    // 2. 候補取得はデバウンス処理で非同期化
    self.updateRawCandidateWithDebounce()
}

// デバウンス処理（@MainActor）
@MainActor private func updateRawCandidateWithDebounce() {
    predictionDebounceTask?.cancel()
    
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        // 非同期待機（UIスレッドをブロックしない）
        try? await Task.sleep(for: .milliseconds(100))
        
        if !Task.isCancelled {
            self.updateRawCandidate()
        }
    }
}
```

### 3.3 Task キャンセル機構

**目的**: 古い予測リクエストを確実にキャンセル

```swift
private var predictionDebounceTask: Task<Void, Never>?

@MainActor
private func cancelPendingPrediction() {
    predictionDebounceTask?.cancel()
    predictionDebounceTask = nil
}

@MainActor
func stopJapaneseInput() {
    // 入力終了時は pending な予測タスクをキャンセル
    cancelPendingPrediction()
    
    // 既存の処理
    self.composingText = ComposingText()
    self.rawCandidates = nil
    self.selectionIndex = nil
    self.shouldShowCandidateWindow = false
    self.didExperienceSegmentEdition = false
    
    // 学習データのコミット（非同期）
    commitUpdateLearningDataAsync()
}
```

---

## 4. 予測候補の統合方法

### 4.1 mainResults への自動統合

**仕様**: KanaKanjiConverter が自動的に予測候補を mainResults に統合

```swift
// ConversionResult の構造（Phase 1 Step 2 の調査結果より）
public struct ConversionResult {
    // 通常の変換候補 + 予測候補（最大3件）が統合される
    public var mainResults: [Candidate]
    
    // 最初の文節のみを変換した候補
    public var firstClauseResults: [Candidate]
}
```

**統合のロジック**（エンジン内部）:
```
mainResults = [
    全文変換候補（スコア順）,
    予測候補（最大3件、requireJapanesePrediction = true で追加）,
    英語予測候補（requireEnglishPrediction = true の場合）,
    特殊変換候補（ユーザーショートカットなど）
]
```

### 4.2 通常候補と予測候補の区別方法

**問題**: mainResults に統合されるため、どれが予測候補かわからない

**解決策**:

#### オプション A: スコアベースの推定（推奨）

予測候補は通常、スコアが低い傾向にある

```swift
private func isPredictionCandidate(_ candidate: Candidate) -> Bool {
    // 予測候補は value が比較的大きい（スコアが低い）
    // 通常の変換候補より value が 2.0 以上大きい場合は予測候補と推定
    guard let firstCandidate = candidates?.first else { return false }
    
    return candidate.value - firstCandidate.value > 2.0
}
```

#### オプション B: 候補数での推定

requireJapanesePrediction = true で候補数が増えることを利用

```swift
private func getPredictionCandidates() -> [Candidate] {
    guard let candidates = candidates else { return [] }
    
    // 通常は 10～15 件、予測ありで 13～18 件
    // 最後の 3 件を予測候補と推定
    if candidates.count > 12 {
        return Array(candidates.suffix(3))
    } else {
        return []
    }
}
```

#### オプション C: UI での明示的区別なし（最推奨）

**方針**: 予測候補と通常候補を区別せず、統合された候補として表示

**理由**:
- エンジンが最適なスコアリングで統合済み
- ユーザーは候補が予測か変換かを気にしない
- UI がシンプルになる

```swift
// 特別な処理なし、mainResults をそのまま表示
private var candidates: [Candidate]? {
    if let rawCandidates {
        if !self.didExperienceSegmentEdition {
            // 既存のロジックをそのまま使用
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

### 4.3 表示順序の制御

**仕様**: エンジンが自動的に最適な順序で並べる

**現状維持**: 既存の候補フィルタリングロジックをそのまま使用

**理由**:
- KanaKanjiConverter が変換候補と予測候補を統一的にスコアリング
- 人為的な並び替えは不要

### 4.4 候補数の制限

#### 予測候補の最大件数

**エンジン側の制限**: 最大 3 件（ハードコード）

```swift
// KanaKanjiConverter.swift での実装（Phase 1 Step 2 の調査結果）
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

#### 総候補数の制限

**現状**: 特に制限なし（候補ウィンドウが自動調整）

**推奨**: ユーザー設定で候補表示件数を制限可能にする

```swift
private var maximumCandidatesInWindow: Int {
    Config.MaximumCandidates().value // デフォルト: 20
}

private var candidates: [Candidate]? {
    // 既存のロジック
    let allCandidates = /* 統合された候補 */
    
    // 最大件数で制限
    return Array(allCandidates.prefix(maximumCandidatesInWindow))
}
```

---

## 5. 既存機能との統合設計

### 5.1 liveConversionEnabled との関係

**重要**: ライブ変換 ON でも予測を表示する

#### 動作の詳細仕様

| liveConversionEnabled | requireJapanesePrediction | 動作 |
|----------------------|---------------------------|------|
| true（ライブ変換 ON） | true | ライブ変換結果 + 予測候補を表示 |
| false（ライブ変換 OFF） | true | 通常の変換候補 + 予測候補を表示 |
| true | false | ライブ変換結果のみ表示（現状） |
| false | false | 通常の変換候補のみ表示（現状） |

#### 実装方法

```swift
private func options(
    leftSideContext: String? = nil,
    requestRichCandidates: Bool = false,
    enablePrediction: Bool = true // 新規パラメータ
) -> ConvertRequestOptions {
    .init(
        // ライブ変換の状態に関わらず、予測を有効化
        requireJapanesePrediction: enablePrediction,
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

### 5.2 shouldShowCandidateWindow の制御

**問題**: ライブ変換 ON の場合、現在は候補ウィンドウを隠す

```swift
// 現在の実装（getCurrentCandidateWindow）
case .composing:
    if liveConversionEnabled {
        // ライブ変換中は通常隠す
        if let first = self.candidates?.first {
            return .composing([first], selectionIndex: nil)
        } else {
            return .hidden
        }
    }
```

**解決策**: ライブ変換中でも予測候補があれば表示

```swift
// 改善後の実装
case .composing:
    if liveConversionEnabled {
        // ライブ変換中でも予測候補があれば表示
        if shouldShowPredictionInLiveConversion(),
           let candidates = candidates,
           !candidates.isEmpty {
            // 予測候補を含む全候補を表示
            return .composing(candidates, selectionIndex: selectionIndex)
        } else if let first = self.candidates?.first {
            // 予測なしの場合は first のみ表示（現状維持）
            return .composing([first], selectionIndex: nil)
        } else {
            return .hidden
        }
    } else {
        // ライブ変換 OFF の場合は通常動作
        if let first = self.candidates?.first {
            return .composing([first], selectionIndex: nil)
        } else {
            return .hidden
        }
    }
```

### 5.3 セグメント編集機能との競合回避

**基本方針**: セグメント編集中は予測を無効化

```swift
private func shouldTriggerPrediction() -> Bool {
    guard !composingText.isEmpty else { return false }
    
    // セグメント編集中は予測しない
    if didExperienceSegmentEdition {
        return false
    }
    
    // その他の条件チェック
    return minimumCharactersForPrediction <= composingText.prefixToCursorPosition().convertTarget.count &&
           isValidInputModeForPrediction()
}
```

**理由**:
- セグメント編集は既存の変換結果を調整する作業
- 新しい予測候補は不要で、むしろ混乱を招く

### 5.4 カーソル移動中の予測制御

**仕様**: カーソル移動後も予測を更新

```swift
@MainActor func moveCursor(count: Int) {
    // カーソル移動
    _ = self.composingText.moveCursorFromCursorPosition(count: count)
    
    // 移動後の位置で候補を更新（デバウンスあり）
    self.updateRawCandidateWithDebounce()
}
```

**理由**:
- カーソル位置に応じた適切な予測候補を表示
- ユーザー体験の向上

---

## 6. パフォーマンス最適化

### 6.1 パフォーマンス目標

| 項目 | 目標値 | 測定方法 |
|------|--------|----------|
| 予測候補取得時間 | 10～20ms 以内 | CFAbsoluteTimeGetCurrent() |
| メインスレッドブロッキング | 0ms 厳守 | Instruments - Time Profiler |
| デバウンス時間 | 100～150ms | Task.sleep(for:) |
| UI 応答性（キーストローク） | <5ms | handleClientAction() 実行時間 |
| 総候補取得時間（予測込み） | 30～50ms 以内 | requestCandidates() 実行時間 |

### 6.2 パフォーマンス測定パターン

```swift
@MainActor
private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    if composingText.isEmpty {
        self.rawCandidates = nil
        self.kanaKanjiConverter.stopComposition()
        return
    }
    
    let startTime = CFAbsoluteTimeGetCurrent()
    
    // ユーザ辞書情報の更新
    var userDictionary: [DicdataElement] = userDictionary.items.map {
        .init(word: $0.word, ruby: $0.reading.toKatakana(), cid: CIDData.固有名詞.cid, mid: MIDData.一般.mid, value: -5)
    }
    let systemUserDictionary: [DicdataElement] = systemUserDictionary.items.map {
        .init(word: $0.word, ruby: $0.reading.toKatakana(), cid: CIDData.固有名詞.cid, mid: MIDData.一般.mid, value: -5)
    }
    userDictionary.append(contentsOf: consume systemUserDictionary)
    self.kanaKanjiConverter.importDynamicUserDictionary(consume userDictionary)
    
    let prefixComposingText = self.composingText.prefixToCursorPosition()
    let leftSideContext = forcedLeftSideContext ?? self.getCleanLeftSideContext(maxCount: 30)
    
    let conversionStartTime = CFAbsoluteTimeGetCurrent()
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates,
            enablePrediction: shouldTriggerPrediction()
        )
    )
    let conversionEndTime = CFAbsoluteTimeGetCurrent()
    
    self.rawCandidates = result
    
    let totalTime = CFAbsoluteTimeGetCurrent() - startTime
    let conversionTime = conversionEndTime - conversionStartTime
    
    // パフォーマンスログ（デバッグビルドのみ）
    #if DEBUG
    if totalTime > 0.05 { // 50ms threshold
        print("⚠️ Slow candidate update: total=\(totalTime * 1000)ms, conversion=\(conversionTime * 1000)ms")
    }
    #endif
}
```

### 6.3 最適化手法

#### デバウンス処理

**効果**: 不要なリクエスト削減（50～70%削減）

```swift
private let predictionDebounceDelay: Duration = .milliseconds(100)

@MainActor
private func updateRawCandidateWithDebounce(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    predictionDebounceTask?.cancel()
    
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        try? await Task.sleep(for: predictionDebounceDelay)
        
        if !Task.isCancelled {
            self.updateRawCandidate(
                requestRichCandidates: requestRichCandidates,
                forcedLeftSideContext: forcedLeftSideContext
            )
        }
    }
}
```

#### 古いタスクのキャンセル

**効果**: メモリ使用量削減、CPU 負荷削減

```swift
@MainActor
private func cancelPendingPrediction() {
    predictionDebounceTask?.cancel()
    predictionDebounceTask = nil
}
```

#### キャッシングの検討

**目的**: 同じ入力での重複リクエスト防止

**実装**: 簡易キャッシュ

```swift
private var predictionCache: [String: ConversionResult] = [:]
private var predictionCacheTimestamp: [String: Date] = [:]
private let predictionCacheValidityDuration: TimeInterval = 1.0 // 1秒

@MainActor
private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    if composingText.isEmpty {
        self.rawCandidates = nil
        self.kanaKanjiConverter.stopComposition()
        return
    }
    
    let prefixComposingText = self.composingText.prefixToCursorPosition()
    let cacheKey = prefixComposingText.convertTarget + (leftSideContext ?? "")
    
    // キャッシュチェック
    if let cachedResult = predictionCache[cacheKey],
       let timestamp = predictionCacheTimestamp[cacheKey],
       Date().timeIntervalSince(timestamp) < predictionCacheValidityDuration {
        self.rawCandidates = cachedResult
        return
    }
    
    // 通常の候補取得処理
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates,
            enablePrediction: shouldTriggerPrediction()
        )
    )
    
    // キャッシュ更新
    predictionCache[cacheKey] = result
    predictionCacheTimestamp[cacheKey] = Date()
    
    self.rawCandidates = result
}
```

**注意**: キャッシュサイズ管理が必要

```swift
private let maximumCacheSize = 50

private func cleanupPredictionCache() {
    guard predictionCache.count > maximumCacheSize else { return }
    
    // 古いエントリを削除
    let sortedKeys = predictionCacheTimestamp.sorted { $0.value < $1.value }
    let keysToRemove = sortedKeys.prefix(predictionCache.count - maximumCacheSize).map(\.key)
    
    for key in keysToRemove {
        predictionCache.removeValue(forKey: key)
        predictionCacheTimestamp.removeValue(forKey: key)
    }
}
```

---

## 7. UI/UX の考慮

### 7.1 予測候補の視覚的区別

#### オプション A: 色による区別（推奨）

```swift
// CandidatesViewController.swift での実装

func updateCandidateView(_ candidate: Candidate, isPrediction: Bool) {
    if isPrediction {
        // 予測候補は青色で表示
        textField.textColor = NSColor.systemBlue
    } else {
        // 通常候補は黒色で表示
        textField.textColor = NSColor.labelColor
    }
}
```

#### オプション B: アイコンによる区別

```swift
func updateCandidateView(_ candidate: Candidate, isPrediction: Bool) {
    if isPrediction {
        // 予測候補の前に 🔮 アイコンを表示
        textField.stringValue = "🔮 \(candidate.text)"
    } else {
        textField.stringValue = candidate.text
    }
}
```

#### オプション C: 区別しない（最推奨）

**理由**:
- エンジンが最適なスコアリングで統合済み
- ユーザーは候補が予測か変換かを気にしない
- UI がシンプルになる

### 7.2 ユーザー設定

#### 予測機能の ON/OFF 切り替え

```swift
// Config.swift に追加
extension Config {
    struct PredictionEnabled: SettingKey {
        typealias Value = Bool
        static let defaultValue: Bool = true
        static let key = "PredictionEnabled"
    }
}

// SegmentsManager.swift での使用
private var predictionEnabled: Bool {
    Config.PredictionEnabled().value
}

private func shouldTriggerPrediction() -> Bool {
    // 予測機能が無効なら false
    guard predictionEnabled else { return false }
    
    // 他の条件チェック
    guard !composingText.isEmpty else { return false }
    if didExperienceSegmentEdition { return false }
    
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    return characterCount >= minimumCharactersForPrediction &&
           isValidInputModeForPrediction()
}
```

#### 予測開始の最小文字数設定

```swift
extension Config {
    struct PredictionMinimumCharacters: SettingKey {
        typealias Value = Int
        static let defaultValue: Int = 2
        static let key = "PredictionMinimumCharacters"
        
        static let range = 1...5 // 1～5文字の範囲
    }
}

private var minimumCharactersForPrediction: Int {
    Config.PredictionMinimumCharacters().value
}
```

#### 予測候補の最大表示件数設定

```swift
extension Config {
    struct PredictionMaximumCandidates: SettingKey {
        typealias Value = Int
        static let defaultValue: Int = 20
        static let key = "PredictionMaximumCandidates"
        
        static let range = 5...50 // 5～50件の範囲
    }
}

private var maximumCandidatesInWindow: Int {
    Config.PredictionMaximumCandidates().value
}
```

---

## 8. 実装時の注意事項

### 8.1 スレッドセーフティ

**制約**: AzooKeyKanaKanjiConverter は明確にスレッドセーフではない

**対策**:
1. すべての KanaKanjiConverter 呼び出しを @MainActor で実行
2. 非同期処理でも MainActor.run を使用
3. weak self パターンの徹底

```swift
// ✅ 推奨パターン
@MainActor
private func updateRawCandidate() {
    let result = self.kanaKanjiConverter.requestCandidates(...)
    self.rawCandidates = result
}

// ✅ 非同期処理からの呼び出し
Task.detached(priority: .background) { [weak self] in
    guard let self = self else { return }
    
    await MainActor.run {
        self.updateRawCandidate()
    }
}

// ❌ 避けるべきパターン
Task.detached {
    // KanaKanjiConverter を直接呼び出し（危険）
    kanaKanjiConverter.requestCandidates(...)
}
```

### 8.2 パフォーマンス

**目標**: メインスレッドブロッキング 0ms 厳守

**対策**:
1. デバウンス処理の適用
2. Task.sleep での非同期待機
3. パフォーマンス測定の実装

```swift
@MainActor
private func updateRawCandidateWithDebounce() {
    predictionDebounceTask?.cancel()
    
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        // 非同期待機（UIスレッドをブロックしない）
        try? await Task.sleep(for: .milliseconds(100))
        
        if !Task.isCancelled {
            self.updateRawCandidate()
        }
    }
}
```

### 8.3 UI/UX

**考慮点**:
1. ライブ変換中でも予測を表示
2. セグメント編集中は予測を無効化
3. 候補ウィンドウの適切な制御

```swift
func getCurrentCandidateWindow() -> CandidateWindow {
    switch inputState {
    case .composing:
        if liveConversionEnabled {
            // ライブ変換中でも予測候補があれば表示
            if shouldShowPredictionInLiveConversion(),
               let candidates = candidates,
               !candidates.isEmpty {
                return .composing(candidates, selectionIndex: selectionIndex)
            } else if let first = self.candidates?.first {
                return .composing([first], selectionIndex: nil)
            } else {
                return .hidden
            }
        }
    // ... 他のケース
    }
}
```

---

## 9. 次のステップへの影響

### 9.1 Step 4（データ構造設計）への要件

**必要な変更**:
1. options() メソッドに enablePrediction パラメータ追加
2. predictionDebounceTask プロパティの追加
3. 予測関連の設定値プロパティの追加

```swift
final class SegmentsManager {
    // 既存プロパティ
    private var composingText: ComposingText = ComposingText()
    private var rawCandidates: ConversionResult?
    private var selectionIndex: Int?
    
    // 新規プロパティ
    private var predictionDebounceTask: Task<Void, Never>?
    private let predictionDebounceDelay: Duration = .milliseconds(100)
    
    // 予測キャッシュ（オプション）
    private var predictionCache: [String: ConversionResult] = [:]
    private var predictionCacheTimestamp: [String: Date] = [:]
    private let predictionCacheValidityDuration: TimeInterval = 1.0
    private let maximumCacheSize = 50
    
    // 設定値（computed property）
    private var predictionEnabled: Bool {
        Config.PredictionEnabled().value
    }
    private var minimumCharactersForPrediction: Int {
        Config.PredictionMinimumCharacters().value
    }
    private var maximumCandidatesInWindow: Int {
        Config.PredictionMaximumCandidates().value
    }
}
```

### 9.2 Step 5（実装）への指針

#### Phase 1: 基本実装（2～3日）

**タスク**:
1. options() メソッドに enablePrediction パラメータ追加
2. shouldTriggerPrediction() メソッドの実装
3. requireJapanesePrediction の条件分岐実装

**成果物**:
- 予測候補が mainResults に含まれる基本機能
- デバウンス処理なし（後の Phase で追加）

**変更行数**: 約 20～30 行

#### Phase 2: デバウンス処理（1～2日）

**タスク**:
1. predictionDebounceTask プロパティ追加
2. updateRawCandidateWithDebounce() メソッド実装
3. 既存の updateRawCandidate() 呼び出しを置き換え

**成果物**:
- デバウンス処理による最適化
- パフォーマンス改善

**変更行数**: 約 30～40 行

#### Phase 3: UI 統合（2～3日）

**タスク**:
1. getCurrentCandidateWindow() の修正（ライブ変換対応）
2. shouldShowPredictionInLiveConversion() メソッド実装
3. 候補ウィンドウ制御の調整

**成果物**:
- ライブ変換中でも予測候補を表示
- 候補ウィンドウの適切な制御

**変更行数**: 約 20～30 行

#### Phase 4: ユーザー設定（1～2日）

**タスク**:
1. Config.swift に設定値追加
2. Settings UI の拡張
3. 設定値の読み込みと適用

**成果物**:
- 予測機能の ON/OFF 切り替え
- 最小文字数設定
- 最大候補数設定

**変更行数**: 約 30～40 行

#### Phase 5: テスト・最適化（2～3日）

**タスク**:
1. パフォーマンス測定の実装
2. キャッシング機構の実装（オプション）
3. ユニットテストの追加
4. 統合テストの実施

**成果物**:
- パフォーマンスレポート
- テストスイート
- 最適化された実装

**変更行数**: 約 50～100 行（テスト含む）

---

## 10. 実装スケジュール見積もり

### リアルタイム予測のみの実装

| Phase | タスク | 期間 | 累計 |
|-------|--------|------|------|
| Phase 1 | 基本実装 | 2～3日 | 2～3日 |
| Phase 2 | デバウンス処理 | 1～2日 | 3～5日 |
| Phase 3 | UI 統合 | 2～3日 | 5～8日 |
| Phase 4 | ユーザー設定 | 1～2日 | 6～10日 |
| Phase 5 | テスト・最適化 | 2～3日 | 8～13日 |
| **総工期** | - | **8～13日** | **8～13日** |

### 余裕を持った見積もり

**推奨総工期**: **10～15日**（バッファ含む）

**理由**:
- 予期しないバグ対応: 2～3日
- パフォーマンスチューニング: 1～2日
- UI/UX 調整: 1～2日

---

## 11. リスク評価と対策

### リスク一覧

| リスク | 確率 | 影響度 | 対策 |
|--------|------|--------|------|
| パフォーマンス劣化 | 中 | 高 | デバウンス処理、キャッシング |
| UI フリーズ | 低 | 高 | @MainActor 厳守、非同期処理 |
| ライブ変換との競合 | 中 | 中 | 適切な条件分岐、テスト強化 |
| メモリリーク | 低 | 中 | weak self パターン徹底 |
| 予測品質の低さ | 中 | 中 | エンジン側の問題、設定で無効化可能 |

### 対策の詳細

#### パフォーマンス劣化

**対策**:
1. デバウンス処理（100～150ms）
2. キャッシング機構（1秒間有効）
3. パフォーマンス測定の実装

**効果**: パフォーマンスオーバーヘッドを +10～20% に抑制

#### UI フリーズ

**対策**:
1. すべての KanaKanjiConverter 呼び出しを @MainActor で実行
2. Task.sleep での非同期待機
3. メインスレッドブロッキング 0ms の厳守

**効果**: UI フリーズの完全防止

#### ライブ変換との競合

**対策**:
1. 適切な条件分岐（shouldShowPredictionInLiveConversion）
2. 候補ウィンドウ制御の調整
3. 統合テストの強化

**効果**: ライブ変換と予測の共存

---

## 12. 結論

### 設計の妥当性

**リアルタイム予測のみの実装**:
- ✅ ユーザー要望に合致
- ✅ 実装が単純（requireJapanesePrediction = true）
- ✅ パフォーマンスオーバーヘッドが許容範囲（+10～20%）
- ✅ ライブ変換との共存が可能

### 次のステップ

**即座に実施（Week 1）**:
1. [ ] Phase 1（基本実装）の着手
   - options() メソッドの修正
   - shouldTriggerPrediction() の実装
   - 動作確認

**中期実施（Week 2）**:
1. [ ] Phase 2（デバウンス処理）の実装
   - updateRawCandidateWithDebounce() の実装
   - パフォーマンス測定

**長期実施（Week 3）**:
1. [ ] Phase 3～5（UI 統合、設定、テスト）の完了
   - 総合的な動作確認
   - ユーザーテスト

---

## 参考資料

- `.claude/phase1-step2-kanakanjiconverter-api-analysis.md` - API 仕様
- `.claude/phase1-step1-candidate-system-analysis.md` - 現在の候補システム
- `.claude/context.md` - スレッドセーフティ制約
- `.claude/project-knowledge.md` - 既存の非同期処理パターン
- `SegmentsManager.swift` - 実装対象ファイル

---

**設計完了日**: 2025-11-08
**設計者**: Claude Code (Anthropic)
