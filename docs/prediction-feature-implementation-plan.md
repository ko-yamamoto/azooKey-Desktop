# 予測変換機能 実装計画書

## プロジェクト概要

azooKey-Desktop に予測変換機能を追加する。予測変換とは、入力中未確定文字列をもとに単語レベルで一致する単語を予測し候補として表示する機能。

**例**: 「よそ」まで入力すると「予測」と候補が表示される。

## 実装ステップ（12段階）

### Phase 1: 調査・設計フェーズ

#### 1. 現在の候補取得システムの詳細調査と設計検討

**目的**: 既存の変換候補システムとの統合方法を明確化

**調査項目**:
- `SegmentsManager.updateRawCandidate()` の動作フロー
- `ConversionResult` の構造（`mainResults`, `firstClauseResults`）
- `ConvertRequestOptions` のパラメータと予測変換への影響
- 現在の候補フィルタリングとランキングロジック

**設計検討**:
- 予測候補を `ConversionResult` に統合するか、別構造で管理するか
- 予測変換専用の `PredictionResult` 型の必要性

#### 2. AzooKeyKanaKanjiConverter の予測変換 API の調査

**目的**: 外部ライブラリが提供する予測変換機能の確認

**調査項目**:
- `ConvertRequestOptions.requireJapanesePrediction` の動作確認
- 予測変換用の専用メソッドの有無
- 予測候補の品質と速度のベンチマーク
- スレッドセーフティの確認（MainActor での実行が必要）

**代替案**:
- ライブラリに予測機能がない場合、自前実装の検討
  - 辞書データからの前方一致検索
  - Trie 構造を使った効率的な検索

### Phase 2: 設計フェーズ

#### 3. 予測変換のトリガー条件とタイミングを設計

**目的**: いつ予測候補を表示するかの仕様を決定

**設計項目**:

**トリガー条件**:
- 最小文字数: 2-3 文字以上で予測開始
- 入力モード: ひらがな入力時のみ or カタカナも含む
- 変換モードとの関係: 未確定文字列のみ or 変換中も表示

**タイミング**:
- リアルタイム予測: 文字入力のたびに更新
- デバウンス処理: 入力停止後 100-200ms で予測開始
- パフォーマンス考慮: 非同期で候補取得、UI ブロックを防ぐ

**既存機能との統合**:
- `liveConversionEnabled` との関係性
- `shouldShowCandidateWindow` の制御方法

#### 4. 予測変換専用のデータ構造と状態管理を設計

**目的**: 予測候補を効率的に管理する仕組みの設計

**データ構造**:
```swift
// 予測候補の管理構造
struct PredictionResult {
    let predictions: [Candidate]
    let inputContext: String
    let timestamp: Date
}

// SegmentsManager への追加プロパティ
private var predictionCandidates: [Candidate] = []
private var predictionEnabled: Bool = true
private var lastPredictionInput: String = ""
private let predictionQueue = DispatchQueue(label: "com.azookey.prediction", qos: .userInitiated)
```

**状態管理**:
- 予測候補のキャッシュ管理
- 入力文字列との同期
- 予測候補の有効期限（入力が変わったら無効化）

### Phase 3: コア実装フェーズ

#### 5. 予測候補取得ロジックの実装

**目的**: 入力文字列から予測候補を取得する機能を実装

**実装内容**:
```swift
// SegmentsManager に追加
@MainActor
private func updatePredictionCandidates(input: String) {
    guard predictionEnabled && input.count >= 2 else {
        predictionCandidates = []
        return
    }

    // 非同期で予測候補を取得
    Task.detached(priority: .userInitiated) { [weak self] in
        guard let self = self else { return }

        let options = self.options(leftSideContext: "", requestRichCandidates: false)
        // requireJapanesePrediction を true に設定
        var predictionOptions = options
        predictionOptions.requireJapanesePrediction = true

        let composingText = ComposingText(input)
        let predictions = await MainActor.run {
            self.kanaKanjiConverter.requestCandidates(composingText, options: predictionOptions)
        }

        await MainActor.run {
            self.predictionCandidates = predictions.mainResults
            self.delegate?.predictionCandidatesUpdated()
        }
    }
}
```

#### 6. 予測候補の非同期取得とキャッシング機能の実装

**目的**: UI フリーズを防ぎ、パフォーマンスを最適化

**実装内容**:
- **デバウンス処理**: 入力が落ち着いてから予測開始
- **キャッシング**: 同じ入力での重複リクエストを防ぐ
- **優先度制御**: 古いリクエストをキャンセル、最新のみ処理

```swift
private var predictionTask: Task<Void, Never>?
private var predictionCache: [String: [Candidate]] = [:]

@MainActor
private func debouncedPrediction(input: String) {
    // 既存のタスクをキャンセル
    predictionTask?.cancel()

    // キャッシュチェック
    if let cached = predictionCache[input] {
        self.predictionCandidates = cached
        return
    }

    // デバウンス付きで新しいタスク開始
    predictionTask = Task {
        try? await Task.sleep(for: .milliseconds(150))

        guard !Task.isCancelled else { return }
        await fetchPredictions(input: input)
    }
}
```

#### 7. 予測候補と通常変換候補の統合ロジックを実装

**目的**: 予測候補と変換候補を適切に組み合わせて表示

**設計案**:
- **案 A: 統合表示** - 予測候補を変換候補の先頭に挿入
- **案 B: 分離表示** - 予測候補専用セクションを作成
- **案 C: 状態依存** - 未変換時は予測のみ、変換時は通常候補

```swift
private var combinedCandidates: [Candidate]? {
    guard let conversionCandidates = self.candidates else {
        return predictionEnabled ? predictionCandidates : nil
    }

    // 予測候補と変換候補を統合
    // 重複を除外しつつ、予測候補を優先
    let predictionTexts = Set(predictionCandidates.map { $0.text })
    let filteredConversion = conversionCandidates.filter {
        !predictionTexts.contains($0.text)
    }

    return predictionCandidates + filteredConversion
}
```

### Phase 4: UI 実装フェーズ

#### 8. 候補ウィンドウでの予測候補表示 UI の実装

**目的**: ユーザーに予測候補を視覚的に区別して表示

**実装内容**:
- 予測候補の視覚的マーキング（アイコン、色、フォントスタイル）
- セパレーター挿入（予測候補と変換候補の境界）
- `CandidatesViewController` での表示制御

```swift
// CandidateView.swift に追加
override internal func configureCellView(_ cell: CandidateTableCellView, forRow row: Int) {
    let candidate = candidates[row]

    // 予測候補かどうかを判定
    let isPrediction = row < predictionCandidateCount

    if isPrediction {
        cell.candidateTextField.textColor = .systemBlue
        cell.candidateTextField.stringValue = "📝 " + candidate.text
    } else {
        cell.candidateTextField.textColor = .labelColor
        cell.candidateTextField.stringValue = candidate.text
    }
}
```

### Phase 5: 学習・最適化フェーズ

#### 9. 予測変換の学習データ統合の実装

**目的**: ユーザーの入力履歴から予測精度を向上

**実装内容**:
- よく使う単語の優先表示
- 最近入力した単語の優先度向上
- `updateLearningDataAsync()` との統合

```swift
@MainActor
func predictionCandidateCommitted(_ candidate: Candidate) {
    // 予測候補の確定を学習データに反映
    self.updateLearningDataAsync(candidate)

    // 予測候補のスコアを更新（頻度カウント）
    updatePredictionScore(candidate)
}
```

#### 10. パフォーマンステストと最適化

**目的**: UI フリーズや遅延がないことを確認

**テスト項目**:
- 予測候補取得の応答時間（目標: < 100ms）
- メインスレッドブロッキングの測定（目標: 0ms）
- メモリ使用量のプロファイリング
- 1000 回の連続入力でのストレステスト

**最適化項目**:
- キャッシュサイズの調整
- 候補数の制限（最大 20-30 件）
- デバウンス時間の微調整

### Phase 6: テスト・設定フェーズ

#### 11. ユニットテストとインテグレーションテストの作成

**目的**: 機能の正確性と回帰防止を保証

**テストケース**:
- 予測候補取得の正確性テスト
- 非同期処理の並行性テスト
- キャッシュの有効性テスト
- 学習データ統合のテスト
- UI 表示の正確性テスト

#### 12. 設定項目の追加

**目的**: ユーザーが予測変換を制御できるようにする

**設定項目**:
- 予測変換の有効/無効切り替え
- 予測候補の最大表示件数
- 予測開始の最小文字数
- 予測候補の表示スタイル（アイコン、色など）

```swift
// Config.swift に追加
class PredictionEnabled: ConfigItem<Bool> {
    override var key: String { "prediction_enabled" }
    override var defaultValue: Bool { true }
}

class PredictionMinCharacters: ConfigItem<Int> {
    override var key: String { "prediction_min_characters" }
    override var defaultValue: Int { 2 }
}
```

## 技術的考慮事項

### スレッドセーフティ
- **重要**: AzooKeyKanaKanjiConverter は MainActor で実行必須
- 予測候補取得も `MainActor.run` でラップする
- 既存の学習データ非同期処理パターンを踏襲

### パフォーマンス目標
- 予測候補取得: < 100ms
- UI 更新遅延: < 50ms
- メインスレッドブロッキング: 0ms
- メモリ増加: < 10MB

### 既存機能との互換性
- ライブ変換機能との共存
- セグメント編集機能との競合回避
- デバッグウィンドウとの統合

## 実装の優先順位

1. Phase 1-2（調査・設計）を完了してから実装開始
2. Phase 3（コア実装）でプロトタイプを作成
3. Phase 4-6 は並行して進行可能

## リスクと対策

### リスク 1: AzooKeyKanaKanjiConverter に予測 API がない
**対策**: 自前で辞書から前方一致検索を実装

### リスク 2: パフォーマンス劣化
**対策**: プロファイリングと段階的最適化、必要に応じて機能制限

### リスク 3: UI/UX の複雑化
**対策**: シンプルな UI から始め、ユーザーフィードバックで改善

## 参考資料

- 既存の非同期処理パターン: `.claude/project-knowledge.md`
- スレッドセーフティ制約: `.claude/context.md`
- プロジェクト構造: `.claude/codebase_structure` (memory)
