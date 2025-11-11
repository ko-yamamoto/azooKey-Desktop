# Phase 2 - Step 4: リアルタイム予測変換のデータ構造と状態管理設計

**作成日**: 2025-11-08
**対象機能**: リアルタイム予測変換（Live Prediction）
**設計方針**: リアルタイム予測のみを実装（確定後予測は不実装）

---

## エグゼクティブサマリー

### 設計決定の概要

本ドキュメントでは、Phase 2 Step 3 で決定した**リアルタイム予測のみ**の実装に必要なデータ構造と状態管理の詳細設計を定義します。

**主要な設計決定**:
1. ✅ **requireJapanesePrediction を条件付きで true に変更**
2. ✅ **デバウンス処理による最適化**（100ms）
3. ✅ **MainActor による完全なスレッドセーフティ保証**
4. ✅ **メモリリーク防止のための weak self パターン徹底**
5. ✅ **既存の候補管理ロジックとの完全な互換性維持**

**実装の複雑性**: 低（総変更行数: 約 100～150 行）

---

## 1. SegmentsManager への追加プロパティ

### 1.1 デバウンス処理用プロパティ

#### 完全な型定義
```swift
// azooKeyMac/InputController/SegmentsManager.swift に追加

final class SegmentsManager {
    // ... 既存のプロパティ ...
    
    // MARK: - Prediction Debounce Properties
    
    /// 予測候補取得のデバウンス処理用 Task
    /// - Note: 新しい入力があるたびにキャンセルされ、新しいタスクが作成される
    /// - Warning: メモリリーク防止のため、Task内では必ず [weak self] を使用すること
    private var predictionDebounceTask: Task<Void, Never>?
    
    /// デバウンス待機時間
    /// - Note: 100ms はパフォーマンスと UX のバランスが最適
    /// - 100ms 未満: 効果が薄い
    /// - 200ms 以上: ユーザーが待たされる感覚
    private let predictionDebounceDelay: Duration = .milliseconds(100)
}
```

**メモリフットプリント推定**:
- `predictionDebounceTask`: 8 bytes（Optional Task 参照）
- `predictionDebounceDelay`: 16 bytes（Duration 構造体）
- **合計**: 24 bytes（無視できる範囲）

**ライフサイクル管理**:
```swift
// Task のキャンセル時機
@MainActor
func stopComposition() {
    // 1. 入力終了時
    cancelPendingPrediction()
    
    // 既存の処理...
    self.composingText.stopComposition()
    self.kanaKanjiConverter.stopComposition()
    self.rawCandidates = nil
}

@MainActor
func deactivate() {
    // 2. IME 非アクティブ化時
    cancelPendingPrediction()
    
    // 既存の処理...
    self.kanaKanjiConverter.stopComposition()
    self.flushLearningDataSafely()
}

@MainActor
private func cancelPendingPrediction() {
    predictionDebounceTask?.cancel()
    predictionDebounceTask = nil
}
```

### 1.2 予測機能の制御用プロパティ

#### Computed Properties による設定値参照
```swift
final class SegmentsManager {
    // ... 既存のプロパティ ...
    
    // MARK: - Prediction Configuration Properties
    
    /// 予測機能の有効/無効
    /// - Returns: ユーザー設定から取得した Bool 値
    /// - Note: Config.PredictionEnabled() を通じて UserDefaults から読み取る
    private var predictionEnabled: Bool {
        Config.PredictionEnabled().value
    }
    
    /// 予測開始の最小文字数
    /// - Returns: ユーザー設定から取得した Int 値（1～5 の範囲）
    /// - Note: デフォルトは 2 文字
    private var predictionMinimumCharacters: Int {
        Config.PredictionMinimumCharacters().value
    }
    
    /// 予測候補の最大表示件数（オプション機能）
    /// - Returns: ユーザー設定から取得した Int 値（5～50 の範囲）
    /// - Note: デフォルトは 20 件
    /// - Warning: この設定は将来的な拡張用。現在は使用していない
    private var predictionMaximumCandidates: Int {
        Config.PredictionMaximumCandidates().value
    }
}
```

**設計の理由**:
- Computed Property を使用することで、設定変更が即座に反映される
- UserDefaults 経由でユーザーが設定を変更可能
- メモリフットプリント: 0 bytes（Computed Property なので）

**パフォーマンス考慮**:
- UserDefaults アクセスは高速（キャッシュされている）
- 頻繁に呼ばれるメソッド内では一時変数にキャッシュ可能

### 1.3 キャッシング用プロパティ（オプション：現時点では不実装）

**実装判断**: 現時点ではキャッシング機構は**実装しない**

**理由**:
1. デバウンス処理で既に十分な最適化が実現される
2. キャッシュ管理のオーバーヘッドが追加される
3. 実装の複雑性が増加する
4. 測定可能なパフォーマンス改善効果が限定的

**将来的な実装の余地**:
```swift
// 将来的に必要になった場合の実装案（現時点では不実装）
/*
// MARK: - Prediction Cache Properties (Future Enhancement)

/// 予測候補のキャッシュ
/// - Key: 入力文字列 + 左文脈のハッシュ
/// - Value: 予測候補の ConversionResult
private var predictionCache: [String: ConversionResult] = [:]

/// キャッシュの有効期限
private let predictionCacheValidityDuration: TimeInterval = 1.0

/// 最後に予測を実行した入力文字列
private var lastPredictionInput: String = ""

/// キャッシュの有効期限時刻
private var cacheExpirationTime: Date?

/// 最大キャッシュサイズ
private let maximumCacheSize = 50
*/
```

---

## 2. options() メソッドの変更

### 2.1 現在の実装（line 183-198）

```swift
private func options(
    leftSideContext: String? = nil, 
    requestRichCandidates: Bool = false
) -> ConvertRequestOptions {
    .init(
        requireJapanesePrediction: false,  // ← 常に false（変更対象）
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

### 2.2 変更後の実装

```swift
private func options(
    leftSideContext: String? = nil,
    requestRichCandidates: Bool = false
) -> ConvertRequestOptions {
    .init(
        // 予測機能を条件付きで有効化
        requireJapanesePrediction: shouldEnablePrediction(),
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

/// 予測機能を有効化すべきかどうかを判定
/// - Returns: true の場合、予測候補を取得する
/// - Note: この判定は高速に実行される（< 1ms）
private func shouldEnablePrediction() -> Bool {
    // 1. ユーザー設定で予測機能が無効の場合
    guard predictionEnabled else { return false }
    
    // 2. 入力が空の場合
    guard !composingText.isEmpty else { return false }
    
    // 3. セグメント編集中の場合
    if didExperienceSegmentEdition { return false }
    
    // 4. カーソル位置までのテキストを取得
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    // 5. 最小文字数を満たしているか
    guard characterCount >= predictionMinimumCharacters else { return false }
    
    // 6. 入力モードが日本語かどうか（将来的な拡張用）
    // 現在は常に true（日本語入力のみサポート）
    // guard isValidInputModeForPrediction() else { return false }
    
    return true
}
```

**変更内容のまとめ**:
- `requireJapanesePrediction: false` → `requireJapanesePrediction: shouldEnablePrediction()`
- `shouldEnablePrediction()` メソッドを新規追加
- 既存のメソッドシグネチャは変更なし（互換性維持）

**パフォーマンス影響**:
- `shouldEnablePrediction()` の実行時間: < 1ms
- 条件判定のみで、重い処理なし

---

## 3. デバウンス処理のメソッド設計

### 3.1 updateRawCandidate() の拡張

#### 既存の実装（line 391-415）
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
    
    // ユーザ辞書情報の更新
    var userDictionary: [DicdataElement] = userDictionary.items.map {
        .init(word: $0.word, ruby: $0.reading.toKatakana(), cid: CIDData.固有名詞.cid, mid: MIDData.一般.mid, value: -5)
    }
    self.appendDebugMessage("userDictionaryCount: \(userDictionary.count)")
    let systemUserDictionary: [DicdataElement] = systemUserDictionary.items.map {
        .init(word: $0.word, ruby: $0.reading.toKatakana(), cid: CIDData.固有名詞.cid, mid: MIDData.一般.mid, value: -5)
    }
    self.appendDebugMessage("systemUserDictionaryCount: \(systemUserDictionary.count)")
    userDictionary.append(contentsOf: consume systemUserDictionary)
    
    self.kanaKanjiConverter.importDynamicUserDictionary(consume userDictionary)
    
    let prefixComposingText = self.composingText.prefixToCursorPosition()
    let leftSideContext = forcedLeftSideContext ?? self.getCleanLeftSideContext(maxCount: 30)
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates
        )
    )
    self.rawCandidates = result
}
```

#### 変更箇所の特定

**変更不要**: 既存の `updateRawCandidate()` メソッドは変更しない

**理由**:
- `options()` メソッドが既に `shouldEnablePrediction()` を呼び出すため
- デバウンス処理は別メソッド `updateRawCandidateWithDebounce()` で実装
- 既存のロジックとの互換性を完全に維持

### 3.2 デバウンス処理の実装

#### 新規メソッド: updateRawCandidateWithDebounce()
```swift
/// デバウンス処理を適用した候補更新
/// - Parameters:
///   - requestRichCandidates: リッチ候補を要求するかどうか
///   - forcedLeftSideContext: 左文脈の上書き（オプション）
/// - Note: 100ms のデバウンス時間を設定
/// - Warning: 既存のタスクは自動的にキャンセルされる
@MainActor
private func updateRawCandidateWithDebounce(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    // 既存のデバウンスタスクをキャンセル
    predictionDebounceTask?.cancel()
    
    // 新しいデバウンスタスクを作成
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        // デバウンス待機（非同期、UIスレッドをブロックしない）
        do {
            try await Task.sleep(for: predictionDebounceDelay)
        } catch {
            // Task.sleep が CancellationError をスローした場合
            // = 新しい入力があってキャンセルされた場合
            return
        }
        
        // キャンセルされていなければ候補を更新
        guard !Task.isCancelled else { return }
        
        // 既存の updateRawCandidate() を呼び出し
        self.updateRawCandidate(
            requestRichCandidates: requestRichCandidates,
            forcedLeftSideContext: forcedLeftSideContext
        )
    }
}
```

**設計の要点**:
1. **weak self パターン**: メモリリーク防止
2. **@MainActor**: スレッドセーフティ保証
3. **do-catch による CancellationError ハンドリング**: 明示的なエラー処理
4. **Task.isCancelled チェック**: 二重チェックで安全性向上

### 3.3 呼び出し箇所の変更

#### 既存の updateRawCandidate() 呼び出し箇所

**対象メソッド**:
- `insertAtCursorPosition(_:inputStyle:)` - line 273
- `insertAtCursorPosition(pieces:inputStyle:)` - line 282
- `editSegment(count:)` - line 313
- `deleteBackwardFromCursorPosition(count:)` - line 328

#### 変更方針

**基本方針**: ユーザー入力による呼び出しのみデバウンス処理を適用

```swift
@MainActor
func insertAtCursorPosition(_ string: String, inputStyle: InputStyle) {
    self.composingText.insertAtCursorPosition(string, inputStyle: inputStyle)
    self.lastOperation = .insert
    self.shouldShowCandidateWindow = !self.liveConversionEnabled
    
    // 変更: デバウンス処理を適用
    self.updateRawCandidateWithDebounce()
}

@MainActor
func insertAtCursorPosition(pieces: [InputPiece], inputStyle: InputStyle) {
    self.composingText.insertAtCursorPosition(pieces.map { .init(piece: $0, inputStyle: inputStyle) })
    self.lastOperation = .insert
    self.shouldShowCandidateWindow = !self.liveConversionEnabled
    
    // 変更: デバウンス処理を適用
    self.updateRawCandidateWithDebounce()
}

@MainActor
func editSegment(count: Int) {
    // ... 既存のロジック ...
    
    self.lastOperation = .editSegment
    self.didExperienceSegmentEdition = true
    self.shouldShowCandidateWindow = true
    self.selectionIndex = nil
    
    // 変更なし: セグメント編集は即座に更新（デバウンスなし）
    self.updateRawCandidate()
}

@MainActor
func deleteBackwardFromCursorPosition(count: Int = 1) {
    // ... 既存のロジック ...
    
    self.composingText.deleteBackwardFromCursorPosition(count: count)
    self.lastOperation = .delete
    self.shouldShowCandidateWindow = !self.liveConversionEnabled
    
    // 変更: デバウンス処理を適用
    self.updateRawCandidateWithDebounce()
}
```

**デバウンス適用の判断基準**:
- ✅ **適用する**: ユーザーの連続入力（insert, delete）
- ❌ **適用しない**: 明示的な変換操作（editSegment, update）

**理由**:
- 連続入力中は頻繁な候補更新が不要
- 明示的な操作では即座の反応が期待される

---

## 4. 予測条件判定メソッドの詳細設計

### 4.1 shouldEnablePrediction() の完全実装

```swift
/// 予測機能を有効化すべきかどうかを判定
/// - Returns: true の場合、予測候補を取得する
/// - Complexity: O(1) - 高速な条件判定のみ
/// - Note: この判定は updateRawCandidate() のたびに実行される
private func shouldEnablePrediction() -> Bool {
    // 1. ユーザー設定で予測機能が無効の場合
    // - UserDefaults から読み取り（キャッシュされているため高速）
    guard predictionEnabled else { return false }
    
    // 2. 入力が空の場合
    // - composingText.isEmpty は O(1) の判定
    guard !composingText.isEmpty else { return false }
    
    // 3. セグメント編集中の場合
    // - セグメント編集中は予測を無効化（UX の観点から）
    if didExperienceSegmentEdition { return false }
    
    // 4. カーソル位置までのテキストを取得
    // - prefixToCursorPosition() は O(n) だが、n は通常小さい（< 50文字）
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    // 5. 最小文字数を満たしているか
    // - デフォルト: 2文字以上
    // - ユーザー設定で 1～5 文字に変更可能
    guard characterCount >= predictionMinimumCharacters else { return false }
    
    // 6. カーソルが末尾にあるかどうか（オプション判定）
    // - 現在は判定していないが、将来的な拡張用にコメントで残す
    /*
    // カーソルが末尾にない場合は予測を無効化
    guard composingText.convertTargetCursorPosition == composingText.convertTarget.count else {
        return false
    }
    */
    
    // 7. 入力モードの判定（将来的な拡張用）
    // - 現在は日本語入力のみサポート
    // - 英語予測は requireEnglishPrediction で対応
    /*
    guard isValidInputModeForPrediction() else { return false }
    */
    
    return true
}
```

**パフォーマンス分析**:
- **最良ケース**: 1～2 チェック（predictionEnabled が false）→ < 0.1ms
- **通常ケース**: 5～6 チェック → < 1ms
- **最悪ケース**: すべてのチェック → < 1ms

**最適化の余地**:
- 現時点では最適化不要（十分に高速）
- 将来的にボトルネックになった場合は、結果をキャッシュすることも可能

### 4.2 将来的な拡張用メソッド（現時点では未実装）

```swift
// 将来的な拡張用メソッド（現時点では実装しない）

/// 入力モードが予測に適しているかどうかを判定
/// - Returns: true の場合、予測を実行可能
/// - Note: 現在は常に true を返す（日本語入力のみサポート）
/*
private func isValidInputModeForPrediction() -> Bool {
    // 将来的には入力モード（ひらがな/カタカナ/英数字）を判定
    // 現在は日本語入力のみなので常に true
    return true
}
*/

/// 入力内容が予測に適しているかどうかを判定
/// - Returns: true の場合、予測を実行可能
/// - Note: 記号のみ、数字のみの入力を除外
/*
private func isValidContentForPrediction() -> Bool {
    let prefixText = composingText.prefixToCursorPosition()
    let target = prefixText.convertTarget
    
    // 空白のみの場合
    guard !target.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty else {
        return false
    }
    
    // 全て記号または数字の場合
    let isOnlySymbols = target.allSatisfy { char in
        char.isSymbol || char.isPunctuation || char.isNumber
    }
    
    return !isOnlySymbols
}
*/
```

---

## 5. 状態管理の設計

### 5.1 予測候補のライフサイクル

#### リアルタイム予測の場合の状態遷移

```
┌─────────────────────────────────────────────────────────┐
│ 状態 1: 入力開始（Empty State）                         │
│ - composingText.isEmpty = true                          │
│ - rawCandidates = nil                                   │
│ - predictionDebounceTask = nil                          │
└─────────────────────────────────────────────────────────┘
                        │
                        │ ユーザー入力（insertAtCursorPosition）
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 状態 2: デバウンス待機中（Debouncing）                  │
│ - composingText.isEmpty = false                         │
│ - rawCandidates = 前回の結果 or nil                     │
│ - predictionDebounceTask = Task（実行中）               │
│ - 100ms 待機中...                                       │
└─────────────────────────────────────────────────────────┘
                        │
                        │ デバウンス完了 or キャンセル
                        ▼
        ┌───────────────┴───────────────┐
        │                               │
        │ キャンセルされた             │ デバウンス完了
        ▼                               ▼
┌───────────────────┐     ┌─────────────────────────────┐
│ 状態 3a: リセット │     │ 状態 3b: 候補取得実行       │
│ - 新しい入力待ち  │     │ - updateRawCandidate() 実行 │
│ - Task = nil      │     │ - KanaKanjiConverter 呼び出し│
└───────────────────┘     └─────────────────────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────────┐
                          │ 状態 4: 候補表示中          │
                          │ - rawCandidates = 取得結果  │
                          │ - mainResults に予測候補含む│
                          │ - 候補ウィンドウ表示        │
                          └─────────────────────────────┘
                                        │
                                        │ ユーザー選択 or 確定
                                        ▼
                          ┌─────────────────────────────┐
                          │ 状態 5: 候補確定            │
                          │ - 学習データ更新            │
                          │ - composingText クリア      │
                          │ - rawCandidates = nil       │
                          └─────────────────────────────┘
```

### 5.2 予測候補と通常候補の統合

#### ConversionResult の構造

```swift
// KanaKanjiConverter からの戻り値
public struct ConversionResult {
    // 通常の変換候補 + 予測候補（最大3件）が自動統合される
    public var mainResults: [Candidate]
    
    // 最初の文節のみの変換候補
    public var firstClauseResults: [Candidate]
}
```

**重要**: 予測候補と通常候補の区別は不要

**理由**:
1. KanaKanjiConverter が最適なスコアリングで自動統合
2. ユーザーは候補が予測か変換かを気にしない
3. UI がシンプルになる

#### 既存の candidates プロパティ（line 339-358）

**変更不要**: 既存のロジックをそのまま使用

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
                // firstClauseCandidateがmainResultsと同じサイズの場合
                return rawCandidates.mainResults
            } else {
                // 変換範囲がエディットされていない場合
                let seenAsFirstClauseResults = rawCandidates.firstClauseResults.mapSet(transform: \.text)
                return rawCandidates.firstClauseResults + rawCandidates.mainResults.filter {
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

**予測候補の統合方法**:
- `requireJapanesePrediction = true` の場合、KanaKanjiConverter が自動的に予測候補を `mainResults` に追加
- 予測候補は通常、`mainResults` の最後の方に追加される（スコアが低いため）
- 既存のフィルタリングロジックがそのまま適用される

### 5.3 状態管理の責任分離

#### SegmentsManager の責任範囲

```
┌────────────────────────────────────────────────────────┐
│ SegmentsManager                                        │
│                                                        │
│ 責任範囲:                                              │
│ 1. 入力テキストの管理（composingText）                │
│ 2. 候補取得のトリガー制御                              │
│ 3. デバウンス処理の管理                                │
│ 4. 予測機能の有効/無効判定                             │
│                                                        │
│ 責任範囲外（KanaKanjiConverter が担当）:               │
│ - 予測候補の生成アルゴリズム                           │
│ - 候補のスコアリング                                   │
│ - 予測候補と通常候補の統合                             │
└────────────────────────────────────────────────────────┘
```

**設計原則**:
- SegmentsManager は**いつ**予測するかを制御
- KanaKanjiConverter は**何を**予測するかを決定
- 明確な責任分離により、実装がシンプルになる

---

## 6. スレッドセーフティの詳細設計

### 6.1 MainActor の適用

#### すべての KanaKanjiConverter 呼び出しを MainActor で実行

**制約**: AzooKeyKanaKanjiConverter は明確にスレッドセーフではない

**対策**: すべての呼び出しを @MainActor で包む

```swift
// ✅ 推奨パターン
@MainActor
private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    // MainActor で実行されることが保証される
    let result = kanaKanjiConverter.requestCandidates(
        composingText,
        options: options(
            leftSideContext: leftSideContext,
            requestRichCandidates: requestRichCandidates
        )
    )
    
    // 結果を保存（MainActor 上で実行）
    self.rawCandidates = result
}
```

**既存の実装との整合性**:
- 既存の `updateRawCandidate()` は既に `@MainActor` 修飾されている（line 391）
- 新規メソッド `updateRawCandidateWithDebounce()` も `@MainActor` を適用

### 6.2 weak self パターンの徹底

#### Task 内での weak self パターン

```swift
// ✅ 推奨パターン
predictionDebounceTask = Task { @MainActor [weak self] in
    guard let self = self else { return }
    
    // Task.sleep は CancellationError をスローする可能性がある
    do {
        try await Task.sleep(for: predictionDebounceDelay)
    } catch {
        // キャンセルされた場合は早期リターン
        return
    }
    
    guard !Task.isCancelled else { return }
    
    // self を使用（weak self によりメモリリーク防止）
    self.updateRawCandidate(
        requestRichCandidates: requestRichCandidates,
        forcedLeftSideContext: forcedLeftSideContext
    )
}
```

**メモリリーク防止の仕組み**:
1. `[weak self]` により、Task は SegmentsManager を強参照しない
2. `guard let self = self else { return }` により、SegmentsManager が解放されていたら早期リターン
3. Task が長時間実行されていても、SegmentsManager の解放をブロックしない

### 6.3 Task のライフサイクル管理

#### キャンセル処理のタイミング

```swift
@MainActor
private func cancelPendingPrediction() {
    // 既存のタスクをキャンセル
    predictionDebounceTask?.cancel()
    
    // nil に設定してリソース解放
    predictionDebounceTask = nil
}
```

**呼び出し箇所**:
1. `stopComposition()` - 入力終了時
2. `deactivate()` - IME 非アクティブ化時
3. `updateRawCandidateWithDebounce()` - 新しい入力時（古いタスクをキャンセル）

**キャンセル時の動作**:
```
┌──────────────────────────────────────────────────────┐
│ Task 実行中                                          │
│ - Task.sleep(for: .milliseconds(100)) 待機中        │
└──────────────────────────────────────────────────────┘
                    │
                    │ predictionDebounceTask?.cancel()
                    ▼
┌──────────────────────────────────────────────────────┐
│ Task キャンセル                                      │
│ - Task.sleep が CancellationError をスロー          │
│ - catch ブロックで return                           │
│ - updateRawCandidate() は実行されない               │
└──────────────────────────────────────────────────────┘
```

---

## 7. Config 項目の詳細設計

### 7.1 BoolConfigItem: PredictionEnabled

#### 完全な実装

```swift
// azooKeyMac/Configs/BoolConfigItem.swift に追加

extension Config {
    /// 予測変換機能の有効/無効
    /// - Note: デフォルトは true（有効）
    /// - UI: 設定画面でチェックボックスとして表示
    struct PredictionEnabled: BoolConfigItem {
        static let `default` = true
        static var key: String = "dev.ensan.inputmethod.azooKeyMac.preference.enablePrediction"
    }
}
```

**設定値の永続化**:
- UserDefaults により自動的に保存される
- キー: `dev.ensan.inputmethod.azooKeyMac.preference.enablePrediction`
- 型: Bool

**デフォルト値の選択理由**:
- `true`: 予測機能はユーザーにとって有用なため、デフォルトで有効

### 7.2 IntConfigItem: PredictionMinimumCharacters

#### 完全な実装

```swift
// azooKeyMac/Configs/IntConfigItem.swift に追加

extension Config {
    /// 予測開始の最小文字数
    /// - Note: デフォルトは 2 文字
    /// - Range: 1～5 文字
    /// - UI: 設定画面でスライダーまたはステッパーとして表示
    struct PredictionMinimumCharacters: IntConfigItem {
        static let `default` = 2
        static let key = "dev.ensan.inputmethod.azooKeyMac.preference.predictionMinimumCharacters"
    }
}
```

**検証ロジック（オプション）**:
```swift
extension Config {
    struct PredictionMinimumCharacters: IntConfigItem {
        static let `default` = 2
        static let key = "dev.ensan.inputmethod.azooKeyMac.preference.predictionMinimumCharacters"
        
        // 値の検証（1～5 の範囲に制限）
        var value: Int {
            get {
                if let value = UserDefaults.standard.value(forKey: Self.key) {
                    let intValue = value as? Int ?? Self.default
                    return max(1, min(5, intValue))  // 1～5 に制限
                } else {
                    return Self.default
                }
            }
            nonmutating set {
                let clampedValue = max(1, min(5, newValue))  // 1～5 に制限
                UserDefaults.standard.set(clampedValue, forKey: Self.key)
            }
        }
    }
}
```

**設定値の意味**:
- 1 文字: 最も早く予測開始（候補が多すぎる可能性）
- 2 文字: バランスが良い（推奨）
- 3～5 文字: 予測開始が遅い（候補が少なくなる）

### 7.3 IntConfigItem: PredictionMaximumCandidates（将来的な拡張用）

#### 完全な実装（現時点では未使用）

```swift
// azooKeyMac/Configs/IntConfigItem.swift に追加（将来的な拡張用）

extension Config {
    /// 予測候補の最大表示件数
    /// - Note: デフォルトは 20 件
    /// - Range: 5～50 件
    /// - UI: 設定画面でスライダーまたはステッパーとして表示
    /// - Warning: 現時点では使用していない（将来的な拡張用）
    struct PredictionMaximumCandidates: IntConfigItem {
        static let `default` = 20
        static let key = "dev.ensan.inputmethod.azooKeyMac.preference.predictionMaximumCandidates"
        
        // 値の検証（5～50 の範囲に制限）
        var value: Int {
            get {
                if let value = UserDefaults.standard.value(forKey: Self.key) {
                    let intValue = value as? Int ?? Self.default
                    return max(5, min(50, intValue))  // 5～50 に制限
                } else {
                    return Self.default
                }
            }
            nonmutating set {
                let clampedValue = max(5, min(50, newValue))  // 5～50 に制限
                UserDefaults.standard.set(clampedValue, forKey: Self.key)
            }
        }
    }
}
```

**現時点での使用状況**:
- ❌ **未使用**: 候補数の制限は KanaKanjiConverter が自動的に行うため
- ✅ **将来的な拡張の余地**: ユーザーが候補数を制限したい場合に使用可能

---

## 8. 実装時のチェックリスト

### 8.1 コンパイル時の注意点

#### 必須の import 文
```swift
// azooKeyMac/InputController/SegmentsManager.swift の冒頭に既存
import Core
import Foundation
import InputMethodKit
import KanaKanjiConverterModuleWithDefaultDictionary

// 追加の import は不要
```

#### 型の整合性チェック
- [ ] `predictionDebounceTask: Task<Void, Never>?` の型が正しい
- [ ] `predictionDebounceDelay: Duration` の型が正しい
- [ ] `Config.PredictionEnabled().value` が `Bool` を返す
- [ ] `Config.PredictionMinimumCharacters().value` が `Int` を返す

#### @MainActor の適切な使用
- [ ] `updateRawCandidateWithDebounce()` に `@MainActor` が付与されている
- [ ] Task クロージャ内に `@MainActor` が付与されている
- [ ] `shouldEnablePrediction()` は `@MainActor` 不要（純粋な判定関数）

### 8.2 テストすべき項目

#### 基本機能テスト
- [ ] **予測機能の ON/OFF 切り替え**
  - Config.PredictionEnabled を false に設定
  - 予測候補が表示されないことを確認
  
- [ ] **最小文字数の設定**
  - Config.PredictionMinimumCharacters を 1, 2, 3 に変更
  - それぞれの文字数で予測が開始されることを確認

- [ ] **デバウンス処理の動作**
  - 高速で連続入力
  - 100ms 経過後に候補が表示されることを確認
  - 連続入力中は候補が更新されないことを確認

#### エッジケーステスト
- [ ] **空入力時の動作**
  - composingText が空の場合、予測が実行されないことを確認
  
- [ ] **セグメント編集中の動作**
  - didExperienceSegmentEdition が true の場合、予測が無効化されることを確認

- [ ] **IME 非アクティブ化時の動作**
  - deactivate() 呼び出し時、predictionDebounceTask がキャンセルされることを確認

#### スレッドセーフティテスト
- [ ] **メモリリークの検証**
  - Instruments - Leaks で検証
  - 長時間の入力後にメモリリークがないことを確認

- [ ] **MainActor 違反の検証**
  - Thread Sanitizer を有効化
  - データ競合が発生しないことを確認

### 8.3 パフォーマンス検証ポイント

#### 測定すべき指標
- [ ] **デバウンス処理のオーバーヘッド**
  - 目標: < 1ms
  - 測定方法: CFAbsoluteTimeGetCurrent()

- [ ] **shouldEnablePrediction() の実行時間**
  - 目標: < 1ms
  - 測定方法: CFAbsoluteTimeGetCurrent()

- [ ] **予測候補取得時間（requireJapanesePrediction = true）**
  - 目標: 10～20ms 以内（+10～20% オーバーヘッド）
  - 測定方法: requestCandidates() の実行時間を測定

#### パフォーマンス測定コード
```swift
#if DEBUG
@MainActor
private func updateRawCandidate(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    let startTime = CFAbsoluteTimeGetCurrent()
    
    // 既存の処理...
    
    let totalTime = CFAbsoluteTimeGetCurrent() - startTime
    
    if totalTime > 0.05 { // 50ms threshold
        print("⚠️ Slow candidate update: \(totalTime * 1000)ms")
    }
}
#endif
```

---

## 9. 次のステップ（Step 5: 実装）への準備

### 9.1 実装する順序

#### Phase 1: Config 設定項目の追加（30分）
```swift
// 1. azooKeyMac/Configs/BoolConfigItem.swift に追加
extension Config {
    struct PredictionEnabled: BoolConfigItem {
        static let `default` = true
        static var key: String = "dev.ensan.inputmethod.azooKeyMac.preference.enablePrediction"
    }
}

// 2. azooKeyMac/Configs/IntConfigItem.swift に追加
extension Config {
    struct PredictionMinimumCharacters: IntConfigItem {
        static let `default` = 2
        static let key = "dev.ensan.inputmethod.azooKeyMac.preference.predictionMinimumCharacters"
    }
}
```

#### Phase 2: SegmentsManager プロパティ追加（30分）
```swift
// azooKeyMac/InputController/SegmentsManager.swift に追加

final class SegmentsManager {
    // MARK: - Prediction Debounce Properties
    private var predictionDebounceTask: Task<Void, Never>?
    private let predictionDebounceDelay: Duration = .milliseconds(100)
    
    // MARK: - Prediction Configuration Properties
    private var predictionEnabled: Bool {
        Config.PredictionEnabled().value
    }
    private var predictionMinimumCharacters: Int {
        Config.PredictionMinimumCharacters().value
    }
}
```

#### Phase 3: shouldEnablePrediction() 実装（30分）
```swift
// azooKeyMac/InputController/SegmentsManager.swift に追加

private func shouldEnablePrediction() -> Bool {
    guard predictionEnabled else { return false }
    guard !composingText.isEmpty else { return false }
    if didExperienceSegmentEdition { return false }
    
    let prefixText = composingText.prefixToCursorPosition()
    let characterCount = prefixText.convertTarget.count
    
    guard characterCount >= predictionMinimumCharacters else { return false }
    
    return true
}
```

#### Phase 4: options() メソッド変更（15分）
```swift
// azooKeyMac/InputController/SegmentsManager.swift の既存メソッドを変更

private func options(
    leftSideContext: String? = nil,
    requestRichCandidates: Bool = false
) -> ConvertRequestOptions {
    .init(
        requireJapanesePrediction: shouldEnablePrediction(),  // ← 変更
        // ... 既存のコード ...
    )
}
```

#### Phase 5: デバウンス処理実装（1時間）
```swift
// azooKeyMac/InputController/SegmentsManager.swift に追加

@MainActor
private func updateRawCandidateWithDebounce(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    predictionDebounceTask?.cancel()
    
    predictionDebounceTask = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        do {
            try await Task.sleep(for: predictionDebounceDelay)
        } catch {
            return
        }
        
        guard !Task.isCancelled else { return }
        
        self.updateRawCandidate(
            requestRichCandidates: requestRichCandidates,
            forcedLeftSideContext: forcedLeftSideContext
        )
    }
}

@MainActor
private func cancelPendingPrediction() {
    predictionDebounceTask?.cancel()
    predictionDebounceTask = nil
}
```

#### Phase 6: 呼び出し箇所の変更（30分）
```swift
// azooKeyMac/InputController/SegmentsManager.swift の既存メソッドを変更

@MainActor
func insertAtCursorPosition(_ string: String, inputStyle: InputStyle) {
    self.composingText.insertAtCursorPosition(string, inputStyle: inputStyle)
    self.lastOperation = .insert
    self.shouldShowCandidateWindow = !self.liveConversionEnabled
    self.updateRawCandidateWithDebounce()  // ← 変更
}

// 他のメソッドも同様に変更
```

#### Phase 7: キャンセル処理の追加（15分）
```swift
// azooKeyMac/InputController/SegmentsManager.swift の既存メソッドを変更

@MainActor
func stopComposition() {
    cancelPendingPrediction()  // ← 追加
    
    // 既存のコード...
    self.composingText.stopComposition()
    self.kanaKanjiConverter.stopComposition()
    self.rawCandidates = nil
}

@MainActor
func deactivate() {
    cancelPendingPrediction()  // ← 追加
    
    // 既存のコード...
    self.kanaKanjiConverter.stopComposition()
    self.flushLearningDataSafely()
}
```

### 9.2 各実装の優先順位

| 優先度 | Phase | 理由 |
|--------|-------|------|
| 1 | Phase 1: Config 追加 | 基盤となる設定システム |
| 2 | Phase 2: プロパティ追加 | データ構造の定義 |
| 3 | Phase 3: shouldEnablePrediction() | 基本ロジックの実装 |
| 4 | Phase 4: options() 変更 | 予測機能の有効化 |
| 5 | Phase 5: デバウンス実装 | パフォーマンス最適化 |
| 6 | Phase 6: 呼び出し箇所変更 | 実際の動作に必要 |
| 7 | Phase 7: キャンセル処理 | リソース管理の安全性 |

**総所要時間見積もり**: 約 3～4 時間

### 9.3 実装時のコードテンプレート

#### テンプレート 1: Config 項目追加
```swift
// ファイル: azooKeyMac/Configs/BoolConfigItem.swift
// 追加位置: extension Config の末尾

extension Config {
    /// [設定項目の説明]
    /// - Note: デフォルトは [デフォルト値]
    /// - UI: [UI表示方法]
    struct [設定項目名]: BoolConfigItem {
        static let `default` = [デフォルト値]
        static var key: String = "dev.ensan.inputmethod.azooKeyMac.preference.[キー名]"
    }
}
```

#### テンプレート 2: デバウンスメソッド
```swift
// ファイル: azooKeyMac/InputController/SegmentsManager.swift
// 追加位置: private メソッド群の末尾

@MainActor
private func [メソッド名]WithDebounce(
    requestRichCandidates: Bool = false,
    forcedLeftSideContext: String? = nil
) {
    [デバウンスタスク]?.cancel()
    
    [デバウンスタスク] = Task { @MainActor [weak self] in
        guard let self = self else { return }
        
        do {
            try await Task.sleep(for: [デバウンス時間])
        } catch {
            return
        }
        
        guard !Task.isCancelled else { return }
        
        self.[元のメソッド](
            requestRichCandidates: requestRichCandidates,
            forcedLeftSideContext: forcedLeftSideContext
        )
    }
}
```

---

## 10. 重要な制約と注意事項

### 10.1 技術的制約

#### AzooKeyKanaKanjiConverter のスレッドセーフティ
- ❌ **スレッドセーフではない**: 複数スレッドからの同時アクセス禁止
- ✅ **対策**: すべての呼び出しを @MainActor で実行
- ✅ **検証**: Thread Sanitizer で競合状態を検出

#### メモリ管理
- ❌ **循環参照のリスク**: Task 内での強参照
- ✅ **対策**: [weak self] パターンの徹底
- ✅ **検証**: Instruments - Leaks でメモリリーク検出

### 10.2 パフォーマンス目標

| 項目 | 目標値 | 測定方法 |
|------|--------|----------|
| デバウンス処理オーバーヘッド | < 1ms | CFAbsoluteTimeGetCurrent() |
| shouldEnablePrediction() 実行時間 | < 1ms | CFAbsoluteTimeGetCurrent() |
| 予測候補取得時間 | +10～20% | requestCandidates() 実行時間 |
| メインスレッドブロッキング | 0ms 厳守 | Instruments - Time Profiler |

### 10.3 既存機能との互換性

#### 完全に互換性を維持すべき項目
- ✅ 既存の候補フィルタリングロジック（`candidates` プロパティ）
- ✅ ライブ変換機能（`liveConversionEnabled`）
- ✅ セグメント編集機能（`didExperienceSegmentEdition`）
- ✅ 学習データ非同期処理（`updateLearningDataAsync()`）

#### 影響を受ける可能性のある項目
- ⚠️ 候補取得パフォーマンス（+10～20% のオーバーヘッド）
- ⚠️ 候補表示タイミング（100ms のデバウンス遅延）

---

## 11. まとめと次のアクション

### 11.1 設計の完成度

**完了した設計項目**:
- ✅ データ構造の完全な定義
- ✅ SegmentsManager への変更内容の明確化
- ✅ デバウンス処理の詳細設計
- ✅ スレッドセーフティとメモリ管理の保証
- ✅ Config 項目の定義
- ✅ 実装時のチェックリスト

**総変更行数見積もり**: 約 100～150 行

### 11.2 次のステップ（Step 5: 実装）

**即座に実施可能**:
1. [ ] Phase 1: Config 設定項目の追加（30分）
2. [ ] Phase 2: SegmentsManager プロパティ追加（30分）
3. [ ] Phase 3: shouldEnablePrediction() 実装（30分）

**中期実施**:
4. [ ] Phase 4: options() メソッド変更（15分）
5. [ ] Phase 5: デバウンス処理実装（1時間）

**最終段階**:
6. [ ] Phase 6: 呼び出し箇所の変更（30分）
7. [ ] Phase 7: キャンセル処理の追加（15分）

**総所要時間**: 約 3～4 時間

### 11.3 実装完了の判定基準

**必須項目**:
- [ ] すべてのコードがコンパイル成功
- [ ] Thread Sanitizer でエラーなし
- [ ] Instruments - Leaks でメモリリークなし
- [ ] 基本機能テストがすべてパス
- [ ] パフォーマンス目標達成

**オプション項目**:
- [ ] ユーザー設定 UI の実装
- [ ] 統合テストの実施
- [ ] パフォーマンスレポートの作成

---

## 参考資料

### 関連ドキュメント
- `.claude/phase2-step3-trigger-timing-design.md` - トリガー条件の詳細仕様
- `.claude/phase1-step2-kanakanjiconverter-api-analysis.md` - API 仕様
- `.claude/project-knowledge.md` - 非同期処理のベストプラクティス

### 実装対象ファイル
- `azooKeyMac/InputController/SegmentsManager.swift` - メインの実装箇所
- `azooKeyMac/Configs/BoolConfigItem.swift` - Bool 型設定項目
- `azooKeyMac/Configs/IntConfigItem.swift` - Int 型設定項目

---

**設計完了日**: 2025-11-08
**設計者**: Claude Code (Anthropic)
**レビュー状態**: 実装準備完了
