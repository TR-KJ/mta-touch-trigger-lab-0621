# CHANGELOG

## v1.0 - 2026-06-21

### Added

- `MTA v2.1` をベースに `MTA Touch Trigger v1` を作成
- 選択足TFの初期値を15分に変更
- 15分MTA接触で `SETUP L / SETUP S` を表示
- 接触足の高値/安値を保存
- 接触足高値/安値を終値でブレイクしたら `ENTRY L / ENTRY S` を表示
- セットアップ有効本数を追加
- トリガーライン表示を追加
- 有効期限切れで `EXPIRE` 表示
- ENTRY成立、期限切れ、MTA方向変化でセットアップ解除

### Fixed

- `zP / zH / zB / zT` の4配列すべてを `n` の最小サイズ計算に含めた

### Verified

- TradingView実機で動作確認済み
- ランタイムエラーなし
- 目視上の挙動に大きな問題なし

## v1.1 - 2026-06-21

### Added

- Touch Trigger専用アラート入力を追加
- `SETUP L` 専用アラートを追加
- `SETUP S` 専用アラートを追加
- `ENTRY L` 専用アラートを追加
- `ENTRY S` 専用アラートを追加
- `alertcondition()` によりTradingViewのアラート作成画面で個別条件を選択可能にした

### Notes

- 既存のMTA発生/終了アラートは維持
- v1.0のSETUP/ENTRY判定ロジックは変更なし
- `strategy.entry` / `strategy.exit` は未実装

## v1.1 - 2026-06-21

### Added

- Touch Trigger専用アラート入力を追加
- `TT SETUP L` の `alertcondition()` を追加
- `TT SETUP S` の `alertcondition()` を追加
- `TT ENTRY L` の `alertcondition()` を追加
- `TT ENTRY S` の `alertcondition()` を追加
- `alert()` による動的メッセージ付きアラートを追加
- SETUPアラートでMTA価格とトリガー価格を通知
- ENTRYアラートでENTRY価格とMTA価格を通知

### Verified

- TradingViewのアラート条件に以下が表示されることを確認
  - `TT SETUP L`
  - `TT SETUP S`
  - `TT ENTRY L`
  - `TT ENTRY S`

### Notes

- 既存のMTA発生/終了アラートは維持
- v1.0のSETUP/ENTRY判定ロジックは変更なし
- strategy版は未実装

## v1.2 - 2026-06-21

### Added

- トリガー方式を選択式に変更
- `Setup Candle Break` を追加
  - 接触足高値/安値ブレイク
- `EMA Reclaim` を追加
  - EMA期間を入力で変更可能
  - EMA線の表示ON/OFFを追加
- `LTF Pivot Break` を追加
  - Pivot Left / Pivot Right を入力で変更可能
  - セットアップ時点の直近ピボット高値/安値を保存してトリガーに使用

### Verified

- `Setup Candle Break` でv1.1同様に動作することを確認
- `EMA Reclaim` でEMA線が表示されることを確認
- EMA期間変更によりENTRY位置が変化することを確認
- `LTF Pivot Break` でPivot L/R変更によりENTRY位置が変化することを確認
- `TT SETUP L`
- `TT SETUP S`
- `TT ENTRY L`
- `TT ENTRY S`
  のアラート条件がTradingView上で表示されることを確認

### Notes

- MTA検出ロジックは変更なし
- SETUP判定ロジックは維持
- 既存のTouch Triggerアラートは維持
- strategy版は未実装
