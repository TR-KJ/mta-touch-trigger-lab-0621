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
