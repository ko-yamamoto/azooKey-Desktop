# リアルタイム予測変換機能 設計ドキュメント

## 概要

このドキュメントは、azooKey-Desktop に実装されたリアルタイム予測変換機能の設計、実装の詳細、および元の実装からの変更内容を記録します。

## 機能の目的

ユーザーが文字を入力している最中に、入力内容から予測される変換候補を候補リストに表示し、タブキーで選択できるようにすることで、入力効率を向上させます。

## 設計方針

### 1. 予測候補とライブ変換の分離

**重要な設計決定**: 予測候補はライブ変換とは独立して動作します。

- **ライブ変換**: 入力中のテキストに対して通常の変換結果を表示（従来通り）
- **予測候補**: 候補リストにのみ表示され、ライブ変換のテキストには影響しない

この分離により、以下のメリットがあります：
- ライブ変換の挙動は従来通り維持される
- 予測候補は候補リストから選択するまで確定されない
- ユーザーは予測候補を無視して通常の入力を続けられる

### 2. 候補の取得タイミング

予測候補は以下の条件をすべて満たした場合に取得されます：

1. 予測機能が有効（`PredictionEnabled` = true）
2. 入力が空でない
3. セグメント編集中でない
4. 入力文字数が最小文字数以上（`PredictionMinimumCharacters` = 1）

### 3. UI の動作

- 候補リストは自動的に表示される（タブキーを押さなくても表示）
- 初期状態では候補は選択されていない（青背景なし）
- タブキーまたはスペースキーを押すと候補選択モードに入り、1番目の候補が選択される
- Shift+Tab で前の候補に移動可能

## 実装の詳細

### データ構造

#### SegmentsManager に追加されたプロパティ

```swift
/// 予測候補専用（候補リストにのみ表示、ライブ変換には反映しない）
private var predictionCandidates: ConversionResult?
```

- `rawCandidates`: 通常の変換候補（予測なし、ライブ変換に使用）
- `predictionCandidates`: 予測変換候補（候補リストに表示）

#### 設定項目

```swift
// BoolConfigItem.swift
struct PredictionEnabled: BoolConfigItem {
    static let `default` = true
    static var key: String = "dev.ensan.inputmethod.azooKeyMac.preference.enablePrediction"
}

// IntConfigItem.swift
struct PredictionMinimumCharacters: IntConfigItem {
    static let `default` = 1
    static let key = "dev.ensan.inputmethod.azooKeyMac.preference.predictionMinimumCharacters"
}
```

### 処理フロー

#### 1. 候補の取得フロー

```
ユーザー入力
    ↓
insertAtCursorPosition()
    ↓
updateRawCandidate()
    ├─→ rawCandidates を取得（requireJapanesePrediction: false）
    │   → ライブ変換に使用
    │
    └─→ shouldEnablePrediction() が true の場合
        predictionCandidates を取得（requireJapanesePrediction: true）
        → 候補リストに表示
```

#### 2. 候補の表示フロー

```
getCurrentCandidateWindow()
    ↓
inputState == .composing の場合
    ↓
liveConversionEnabled && shouldEnablePrediction() の場合
    ↓
candidates を返す（selectionIndex: nil）
    ↓
候補リスト表示（未選択状態）
```

#### 3. 候補の選択フロー

```
タブキー/スペースキー押下
    ↓
.composing 状態から .selecting 状態へ遷移
    ↓
selectionIndex = 0 に設定
    ↓
1番目の候補が選択状態（青背景）で表示
```

### 主要メソッドの実装

#### shouldEnablePrediction()

```swift
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

    return true
}
```

#### updateRawCandidate()

```swift
@MainActor
func updateRawCandidate(forcedLeftSideContext: String? = nil, requestRichCandidates: Bool = false) {
    // ... 既存の処理 ...

    // ライブ変換用の候補（予測なし）
    let result = self.kanaKanjiConverter.requestCandidates(
        prefixComposingText,
        options: options(leftSideContext: leftSideContext, requestRichCandidates: requestRichCandidates)
    )
    self.rawCandidates = result

    // 候補リスト表示用の予測候補（予測あり）
    if self.shouldEnablePrediction() {
        var predictionOptions = options(leftSideContext: leftSideContext, requestRichCandidates: requestRichCandidates)
        predictionOptions.requireJapanesePrediction = true
        let predictionResult = self.kanaKanjiConverter.requestCandidates(prefixComposingText, options: predictionOptions)
        self.predictionCandidates = predictionResult
    } else {
        self.predictionCandidates = nil
    }
}
```

#### candidates（計算プロパティ）

```swift
private var candidates: [Candidate]? {
    if let rawCandidates {
        // 通常の候補を取得
        let baseCandidates: [Candidate]
        if !self.didExperienceSegmentEdition {
            if rawCandidates.firstClauseResults.contains(where: { self.composingText.isWholeComposingText(composingCount: $0.composingCount) }) {
                baseCandidates = rawCandidates.mainResults
            } else {
                let seenAsFirstClauseResults = rawCandidates.firstClauseResults.mapSet(transform: \.text)
                baseCandidates = rawCandidates.firstClauseResults + rawCandidates.mainResults.filter {
                    !seenAsFirstClauseResults.contains($0.text)
                }
            }
        } else {
            baseCandidates = rawCandidates.mainResults
        }

        // 予測候補を先頭に追加（候補リストにのみ表示、ライブ変換には反映されない）
        if let predictionCandidates, !self.didExperienceSegmentEdition {
            let seenAsBaseCandidates = baseCandidates.mapSet(transform: \.text)
            let uniquePredictions = (predictionCandidates.firstClauseResults + predictionCandidates.mainResults).filter {
                !seenAsBaseCandidates.contains($0.text)
            }
            return uniquePredictions + baseCandidates
        } else {
            return baseCandidates
        }
    } else {
        return nil
    }
}
```

#### getCurrentMarkedText()

```swift
case .composing:
    let text = if self.lastOperation == .delete {
        // 削除のあとは常にひらがなを示す
        self.composingText.convertTarget
    } else if self.liveConversionEnabled,
              self.composingText.convertTarget.count > 1,
              let firstCandidate = self.rawCandidates?.mainResults.first {
        // ライブ変換が有効な場合はライブ変換を表示
        // rawCandidatesには予測候補が含まれていないため、通常の変換結果のみが表示される
        firstCandidate.text
    } else {
        // ライブ変換無効、または文字数が少ない場合はひらがなを表示
        self.composingText.convertTarget
    }
    return MarkedText(text: [.init(content: text, focus: .none)], selectionRange: .notFound)
```

#### getCurrentCandidateWindow()

```swift
case .composing:
    if !self.liveConversionEnabled, let firstCandidate = self.rawCandidates?.mainResults.first {
        return .composing([firstCandidate], selectionIndex: 0)
    } else if self.liveConversionEnabled, let candidates, !candidates.isEmpty, self.shouldEnablePrediction() {
        // ライブ変換有効かつ予測候補がある場合は候補ウィンドウを表示（未選択状態）
        return .composing(candidates, selectionIndex: nil)
    } else {
        return .hidden
    }
```

## 元の実装からの変更点

### 1. UserAction の追加

**ファイル**: `Core/Sources/Core/InputUtils/Actions/UserAction.swift`

```swift
// 追加
case shiftTab
```

Shift+Tab での逆方向候補移動をサポート。

### 2. キーマッピングの追加

**ファイル**: `azooKeyMac/InputController/UserAction+getUserAction.swift`

```swift
case 48: // Tab
    if event.modifierFlags.contains(.shift) {
        return .shiftTab
    } else {
        return .tab
    }
```

### 3. InputState の変更

**ファイル**: `Core/Sources/Core/InputUtils/InputState.swift`

#### .composing 状態でのタブ/スペースキー処理

**変更前**:
```swift
case .tab:
    return (.selectNextCandidate, .transition(.selecting))
```

**変更後**:
```swift
case .tab:
    if liveConversionEnabled {
        // ライブ変換有効時は候補選択モードに入る（1番目から）
        return (.enterCandidateSelectionMode, .transition(.selecting))
    } else {
        return (.enterFirstCandidatePreviewMode, .transition(.previewing))
    }
```

これにより、タブキーを押したときに1番目の候補が選択される動作になります。

#### .selecting 状態でのタブキー処理

```swift
case .tab:
    return (.selectNextCandidate, .fallthrough)
case .shiftTab:
    return (.selectPrevCandidate, .fallthrough)
```

### 4. Ctrl+j（ひらがな確定）の修正

**ファイル**: `azooKeyMac/InputController/SegmentsManager.swift`

#### getModifiedRubyCandidate の変更

**変更前**:
```swift
func getModifiedRubyCandidate(_ transform: (String) -> String) -> Candidate {
    let ruby = if let selectedCandidate {
        selectedCandidate.data.map { element in
            element.ruby
        }.joined()
    } else {
        self.composingText.convertTarget
    }
    // ...
}
```

**変更後**:
```swift
func getModifiedRubyCandidate(_ transform: (String) -> String, inputState: InputState) -> Candidate {
    // .composing 状態では候補を使わず、常に入力テキストを使う
    let ruby = if inputState != .composing, let selectedCandidate {
        selectedCandidate.data.map { element in
            element.ruby
        }.joined()
    } else {
        // 選択範囲なしの場合、または .composing 状態の場合は convertTarget を返す
        self.composingText.convertTarget
    }
    // ...
}
```

**理由**: .composing 状態で候補リストが表示されている場合でも、Ctrl+j は入力中のテキスト（例: "へん"）をひらがなで確定すべきであり、予測候補の読み（例: "へんか"）を使うべきではない。

**ファイル**: `azooKeyMac/InputController/azooKeyMacInputController.swift`

呼び出し側も更新:
```swift
case .submitHiraganaCandidate:
    self.submitCandidate(self.segmentsManager.getModifiedRubyCandidate({
        $0.toHiragana()
    }, inputState: self.inputState))
case .submitKatakanaCandidate:
    self.submitCandidate(self.segmentsManager.getModifiedRubyCandidate({
        $0.toKatakana()
    }, inputState: self.inputState))
case .submitHankakuKatakanaCandidate:
    self.submitCandidate(self.segmentsManager.getModifiedRubyCandidate({
        $0.toKatakana().applyingTransform(.fullwidthToHalfwidth, reverse: false)!
    }, inputState: self.inputState))
```

### 5. 候補ウィンドウの表示制御

**ファイル**: `azooKeyMac/InputController/SegmentsManager.swift`

#### insertAtCursorPosition の変更

**変更前**:
```swift
self.shouldShowCandidateWindow = !self.liveConversionEnabled
```

**変更後**:
```swift
self.shouldShowCandidateWindow = !self.liveConversionEnabled || self.shouldEnablePrediction()
```

これにより、ライブ変換が有効でも予測機能が有効な場合は候補ウィンドウが表示されます。

### 6. リソース管理

以下のメソッドで `predictionCandidates` のクリーンアップを追加:

```swift
func deactivate() {
    // ...
    self.predictionCandidates = nil
}

func stopComposition() {
    // ...
    self.predictionCandidates = nil
}

func stopJapaneseInput() {
    // ...
    self.predictionCandidates = nil
}
```

## 設定項目

### PredictionEnabled

- **型**: Bool
- **デフォルト値**: true
- **説明**: 予測機能の有効/無効を切り替え
- **UserDefaults キー**: `dev.ensan.inputmethod.azooKeyMac.preference.enablePrediction`

### PredictionMinimumCharacters

- **型**: Int
- **デフォルト値**: 1
- **説明**: 予測候補を表示し始める最小文字数
- **UserDefaults キー**: `dev.ensan.inputmethod.azooKeyMac.preference.predictionMinimumCharacters`

## パフォーマンスの考慮事項

### 候補取得の最適化

1. **判定の高速化**: `shouldEnablePrediction()` は高速に実行される（< 1ms）
2. **条件付き取得**: 予測機能が無効な場合は `predictionCandidates` を取得しない
3. **非同期実行**: KanaKanjiConverter は @MainActor で実行されるため、UI ブロックを最小化

### メモリ管理

- `predictionCandidates` は不要になったタイミングで nil に設定される
- 入力セッションが終了したときに確実にクリーンアップされる

## 既知の制限事項

1. **セグメント編集時**: セグメント編集中は予測候補が表示されない（意図的な設計）
2. **最小文字数**: 設定された最小文字数未満の入力では予測候補が表示されない

## 今後の拡張可能性

1. **予測精度の向上**: より高度な予測アルゴリズムの導入
2. **学習機能**: ユーザーの入力履歴から予測精度を向上
3. **候補数の制御**: 表示する予測候補の数を設定可能にする
4. **カスタマイズ**: 予測候補の表示位置や表示方法のカスタマイズ

## テスト観点

1. **基本動作**:
   - 1文字入力時に予測候補が表示される
   - タブキーで候補を選択できる
   - Shift+Tab で前の候補に戻れる

2. **ライブ変換との共存**:
   - ライブ変換が正常に動作する
   - 予測候補がライブ変換のテキストに影響しない

3. **Ctrl+j の動作**:
   - 入力中のテキストがひらがなで確定される
   - 予測候補の読みが使われない

4. **候補リストの表示**:
   - 初期状態で候補が選択されていない
   - タブキーで選択状態になる

## 参考情報

### 関連ファイル

- `Core/Sources/Core/InputUtils/Actions/UserAction.swift`
- `Core/Sources/Core/InputUtils/InputState.swift`
- `azooKeyMac/Configs/BoolConfigItem.swift`
- `azooKeyMac/Configs/IntConfigItem.swift`
- `azooKeyMac/InputController/SegmentsManager.swift`
- `azooKeyMac/InputController/UserAction+getUserAction.swift`
- `azooKeyMac/InputController/azooKeyMacInputController.swift`

### コミット履歴

- 初回実装: `94c4afb` - リアルタイム予測変換の基本的機能を実装
- 改善とバグ修正: `3b53574` - 予測変換機能の改善とバグ修正

## まとめ

本機能は、ライブ変換と予測候補を明確に分離することで、既存の入力体験を損なうことなく予測変換機能を追加しています。候補リストの UI 改善（未選択状態の導入）や Ctrl+j の修正により、ユーザーにとって直感的で使いやすい実装となっています。
