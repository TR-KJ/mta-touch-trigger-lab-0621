# Strategy v1.0 Lite 検証結果と v1.1 改善方針

---

## 1. 対象ファイル

```text
src/strategy/mta_touch_trigger_strategy_v1_0_lite.pine
```

---

## 2. バージョン

```text
MTA Touch Trigger Strategy v1.0 Lite
```

---

## 3. 開発目的

`mta_touch_trigger_strategy_v1_0_lite.pine` は、`mta_touch_trigger_strategy_v1_0.pine` のバックテスト負荷を下げるために作成した軽量版。

通常版では、1年程度のバックテストでTradingViewが「レポートを更新中」のまま固まることがあった。

そのため、Lite版では描画・アラート・チャート足MTA計算を削除し、バックテストに必要な最低限の処理だけを残した。

---

## 4. Lite版で削除したもの

```text
チャート足MTA request.security
ZigZag描画
MTAライン描画
SETUPラベル
ENTRYラベル
EXPIREラベル
SL/TPライン描画
alert()
alertcondition()
line / label 管理
```

---

## 5. Lite版で残したもの

```text
15分MTA取得
MTA接触SETUP判定
Setup Candle Break
EMA Reclaim
LTF Pivot Break
strategy.entry
strategy.exit
2%リスク数量計算
固定RR
手数料0.04%
スリッページ2
pyramiding = 0
```

---

## 6. 動作確認結果

### 確認内容

```text
90日バックテスト
1年バックテスト
3年バックテスト
```

### 結果

```text
3年でも問題なく動作
```

### 判定

```text
Lite化は成功
```

通常版で重かった原因は、主に描画・アラート・チャート足MTA計算・line/label管理がstrategy検証時に負荷となっていた可能性が高い。

---

## 7. パラメータテスト結果

Lite版で以下のパラメータを変更して複数テストを実施した。

```text
Trigger Mode
RR
EMA期間
Pivot L/R
ExpireBars
```

### 暫定結果

```text
成績は全体的に不十分
```

### 暫定判断

現行の、

```text
15分MTAに接触
↓
1分足トリガー成立
↓
エントリー
```

という構成では、十分な優位性が確認できなかった。

特に、単純なMTA接触をそのままセットアップとして扱う条件が粗すぎる可能性が高い。

---

## 8. 現行セットアップの問題点

現在のセットアップは以下。

### Long Setup

```text
15分MTAが上昇状態
かつ
1分足が15分MTAに接触
```

### Short Setup

```text
15分MTAが下降状態
かつ
1分足が15分MTAに接触
```

この条件では、

```text
MTAに触れたが反発しない
MTAをそのまま終値で抜ける
ヒゲだけ反応してすぐ逆行する
レンジ内で何度もダマシになる
```

といったケースもすべてセットアップとして拾ってしまう。

そのため、

```text
MTAタッチ = セットアップ確定
```

ではなく、

```text
MTAタッチ = 監視開始
```

として扱う方向へ修正する。

---

## 9. v1.1 改善方針

strategy v1.1 Liteでは、まず大きなロジック改造ではなく、セットアップ品質を上げるための最小修正を行う。

追加候補は以下。

```text
1. MTA守り足フィルター
2. MTA終値ブレイクでセットアップ無効
```

---

## 10. 改善案1：MTA守り足フィルター

### 目的

単にMTAへ接触しただけではなく、接触後にMTAを守った足だけをセットアップとして採用する。

---

### Long条件

現行。

```text
low <= sM
かつ
high >= sM
```

v1.1案。

```text
low <= sM
かつ
close > sM
```

意味。

```text
15分上昇MTAに接触したうえで、
1分足終値がMTAより上で確定
```

つまり、MTAを下抜けて終わった足は除外する。

---

### Short条件

現行。

```text
low <= sM
かつ
high >= sM
```

v1.1案。

```text
high >= sM
かつ
close < sM
```

意味。

```text
15分下降MTAに接触したうえで、
1分足終値がMTAより下で確定
```

つまり、MTAを上抜けて終わった足は除外する。

---

### 入力項目案

```pine
useMtaHoldFilter = input.bool(true, 'MTA守り足フィルター', group = 'Setup Filter')
```

ON/OFF可能にして、現行条件との比較ができるようにする。

---

## 11. 改善案2：MTA終値ブレイクでセットアップ無効

### 目的

セットアップ後、トリガー待ち中にMTAを終値で明確に抜けた場合、そのセットアップを無効化する。

---

### Long Setup待機中

```text
close < setupMta
```

になったらセットアップ解除。

意味。

```text
上昇MTAを守れなかったため、Long待機を終了
```

---

### Short Setup待機中

```text
close > setupMta
```

になったらセットアップ解除。

意味。

```text
下降MTAを守れなかったため、Short待機を終了
```

---

### 入力項目案

```pine
useMtaCloseInvalid = input.bool(true, 'MTA終値ブレイクで無効', group = 'Setup Filter')
```

ON/OFF可能にして、現行条件との比較ができるようにする。

---

## 12. v1.1 Liteで変更しないもの

v1.1 Liteでは、以下は変更しない。

```text
MTA検出ロジック
DevLucem/ZigLib設定
15分MTA取得
Trigger Mode 3種類
RR
2%リスク数量計算
SL = セットアップ足高値/安値
TP = 固定RR
手数料0.04%
スリッページ2
pyramiding = 0
```

理由は、まずセットアップ品質の改善効果だけを確認したいため。

---

## 13. v1.1 Lite テスト方針

v1.1 Liteでは、以下の4パターンを比較する。

| No | MTA守り足フィルター | MTA終値ブレイク無効 | 目的 |
|---:|---|---|---|
| 1 | OFF | OFF | v1.0 Lite基準 |
| 2 | ON | OFF | 守り足のみの効果確認 |
| 3 | OFF | ON | 終値ブレイク無効のみの効果確認 |
| 4 | ON | ON | 両方ONの効果確認 |

---

## 14. 優先テスト条件

まずは、v1.0 Liteで比較した中から、相対的にマシだった条件を使う。

相対的に良い条件が不明な場合は、以下を基準にする。

```text
Trigger Mode：Setup Candle Break
RR：1.0 / 1.3 / 1.5 / 2.0
ExpireBars：60
選択足TF：15分
チャート足：1分
銘柄：USDJPY
期間：3年
```

次に、

```text
EMA Reclaim
LTF Pivot Break
```

でも同じフィルター比較を行う。

---

## 15. 判断基準

v1.1 Liteで見るべき指標。

```text
総トレード数
勝率
PF
最大DD
純利益
平均損益
連敗数
```

判断目安。

```text
PF 0.5未満
→ セットアップ改善だけでは不十分。SL Modeや時間帯フィルターが必要。

PF 0.7〜0.9
→ 改善余地あり。SL ModeやHTF EMAフィルターを検討。

PF 0.95〜1.1
→ 深掘り対象。トリガー・RR・時間帯で検証継続。

PF 1.1以上
→ 優先検証候補。
```

---

## 16. 次の開発ステップ

### Step 1

```text
strategy v1.1 Lite を作成
```

追加内容。

```text
MTA守り足フィルター
MTA終値ブレイクでセットアップ無効
```

---

### Step 2

v1.1 Liteで4パターン比較。

```text
OFF / OFF
ON / OFF
OFF / ON
ON / ON
```

---

### Step 3

改善が見られた場合、次にSL Modeを追加する。

候補。

```text
Setup Candle
MTA Buffer
ATR Buffer
LTF Pivot
```

---

## 17. 暫定結論

現時点では、単純なMTA接触だけをセットアップとする条件では優位性が弱い可能性が高い。

次は、MTAタッチをセットアップ確定ではなく、監視開始として扱い、

```text
触った
↓
守った
↓
トリガー成立
↓
エントリー
```

という構造へ改善する。
