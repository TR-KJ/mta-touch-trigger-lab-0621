# docs/04_strategy_spec_v1.md

# Strategy v1.0 仕様書

---

## 1. バージョン名

```text
MTA Touch Trigger Strategy v1.0
```

---

## 2. 開発目的

`MTA Touch Trigger Indicator v1.2` をベースに、バックテスト可能なstrategy版を作成する。

目的は、以下の手法の成績をTradingView Strategy Testerで確認すること。

```text
15分足MTA接触
↓
1分足でトリガー成立
↓
エントリー
↓
セットアップ足高値/安値をSL
↓
固定RRでTP
```

---

## 3. ベースコード

ベースは以下。

```text
src/indicator/mta_touch_trigger_v1_2.pine
```

strategy版では、MTA検出ロジック、SETUP判定、トリガー判定の基本構造は変更しない。

変更するのは、主に売買命令とリスク計算部分。

---

## 4. strategy宣言

strategy版では、`indicator()` を `strategy()` に変更する。

設定は以下。

```pine
strategy(
     'MTA Touch Trigger Strategy v1.0',
     overlay = true,
     max_lines_count = 500,
     max_labels_count = 500,
     initial_capital = 1000000,
     default_qty_type = strategy.fixed,
     default_qty_value = 1,
     commission_type = strategy.commission.percent,
     commission_value = 0.04,
     slippage = 2,
     pyramiding = 0,
     process_orders_on_close = false,
     calc_on_every_tick = false
)
```

---

## 5. 検証前提

### チャート足

```text
1分足
```

### 上位足MTA

```text
15分足
```

### 初期検証銘柄

```text
USDJPY
```

### 対応方向

```text
買い・売り両方
```

### ピラミッディング

```text
なし
```

```pine
pyramiding = 0
```

---

## 6. エントリー条件

エントリー条件は、インジケーターv1.2と同じく3種類から選択式とする。

```text
Setup Candle Break
EMA Reclaim
LTF Pivot Break
```

---

### 6.1 Long Setup

```text
15分MTAが上昇状態
かつ
1分足が15分MTAに接触
```

条件。

```text
sTr == 1
low <= sM
high >= sM
```

---

### 6.2 Short Setup

```text
15分MTAが下降状態
かつ
1分足が15分MTAに接触
```

条件。

```text
sTr == -1
low <= sM
high >= sM
```

---

## 7. トリガー方式

### 7.1 Setup Candle Break

#### Long

```text
Long Setup発生後、
1分足終値がセットアップ足高値を上抜け
```

#### Short

```text
Short Setup発生後、
1分足終値がセットアップ足安値を下抜け
```

---

### 7.2 EMA Reclaim

EMA期間を入力で変更可能にする。

#### Long

```text
Long Setup発生後、
1分足終値がEMAを下から上へ回復
```

条件。

```text
close > EMA
close[1] <= EMA[1]
```

#### Short

```text
Short Setup発生後、
1分足終値がEMAを上から下へ割り込み
```

条件。

```text
close < EMA
close[1] >= EMA[1]
```

---

### 7.3 LTF Pivot Break

Pivot Left / Right を入力で変更可能にする。

#### Long

```text
Long Setup発生時点で保存した直近1分ピボット高値を、
1分足終値が上抜け
```

#### Short

```text
Short Setup発生時点で保存した直近1分ピボット安値を、
1分足終値が下抜け
```

---

## 8. エントリー命令

### Long

```pine
strategy.entry('Long', strategy.long, qty = longQty)
```

### Short

```pine
strategy.entry('Short', strategy.short, qty = shortQty)
```

エントリー判定は確定足ベースで行う。

```pine
barstate.isconfirmed
```

`process_orders_on_close = false` のため、実際の約定は次足始値想定。

---

## 9. 損切り

損切りはセットアップ足を基準にする。

### Long SL

```text
セットアップ足安値
```

```pine
longStopPrice = setupLow
```

### Short SL

```text
セットアップ足高値
```

```pine
shortStopPrice = setupHigh
```

---

## 10. 利確

利確は固定RRで設定する。

### 初期値

```text
RR 1.5
```

入力で変更可能。

```pine
rr = input.float(1.5, 'RR', minval = 0.1, step = 0.1, group = 'Strategy Risk')
```

---

### Long TP

```text
Long Entry価格 + リスク幅 × RR
```

```pine
longLimitPrice = longEntryPriceForCalc + longRiskDistance * rr
```

---

### Short TP

```text
Short Entry価格 - リスク幅 × RR
```

```pine
shortLimitPrice = shortEntryPriceForCalc - shortRiskDistance * rr
```

---

## 11. 出口命令

出口は必ず `strategy.exit()` で設定する。

### Long

```pine
strategy.exit('Long Exit', from_entry = 'Long', stop = longStopPrice, limit = longLimitPrice)
```

### Short

```pine
strategy.exit('Short Exit', from_entry = 'Short', stop = shortStopPrice, limit = shortLimitPrice)
```

出口の書き忘れは禁止。

---

## 12. 数量計算

strategy v1.0では、1回の損失額が現在資金の指定％になるように数量を自動計算する。

### 初期リスク

```text
2%
```

入力で変更可能。

```pine
riskPct = input.float(2.0, '1回リスク％', minval = 0.1, step = 0.1, group = 'Strategy Risk')
```

---

### 計算式

```text
riskAmount = strategy.equity × riskPct / 100
stopDistance = abs(entryPrice - stopPrice)
qty = riskAmount / (stopDistance × syminfo.pointvalue)
```

Pine内では以下のように計算する。

```pine
riskAmount = strategy.equity * riskPct / 100.0
qty = riskAmount / (stopDistance * syminfo.pointvalue)
```

---

## 13. 最小SL幅

SL幅が極端に小さい場合、数量が異常に大きくなるため、最小SL幅を設定する。

```pine
minRiskTick = input.int(1, '最小SL幅 ticks', minval = 1, group = 'Strategy Risk')
minStopDistance = syminfo.mintick * minRiskTick
```

SL幅が `minStopDistance` 未満の場合、エントリーしない。

---

## 14. セットアップ解除条件

以下の場合、セットアップ状態を解除する。

```text
ENTRY成立
有効期限切れ
MTA方向変化
MTA終了
```

---

## 15. アラート

strategy版では、以下のalertconditionを使用する。

```text
TT SETUP L
TT SETUP S
ST ENTRY L
ST ENTRY S
```

インジケーター版の `TT ENTRY L / TT ENTRY S` は、strategy版では実際に注文条件を満たしたものとして `ST ENTRY L / ST ENTRY S` に分ける。

---

## 16. 実機確認項目

strategy版作成後、TradingView上で以下を確認する。

```text
1. コンパイルエラーなし
2. ランタイムエラーなし
3. Strategy Testerにトレードが表示される
4. Long / Short 両方が入る
5. SL / TP が設定される
6. RR変更でTP位置が変わる
7. riskPct変更で数量が変わる
8. Setup Candle Break / EMA Reclaim / LTF Pivot Break が選択できる
9. 手数料0.04%が反映されている
10. スリッページ2が反映されている
```

---

## 17. バックテスト時の注意

以下を必ず確認する。

```text
最低50取引以上で判断する
前半期間 / 後半期間に分けて確認する
上昇局面 / 下降局面 / 横ばい局面を含める
RR 1.0 / 1.3 / 1.5 / 2.0 を比較する
EMA期間 10 / 20 / 50 を比較する
Pivot L/R 3/3、5/5、10/10 を比較する
過剰最適化に注意する
```

---

## 18. このstrategyの限界

### ZigZagは後追いで確定する

DevLucem/ZigLibのZigZagは、ピボットがその場で即確定するわけではない。

後続足の動きによって高値/安値が確定する。

そのため、MTAも一定の遅れを伴って確定する。

---

### MTAは構造確定後に出る

MTAは価格がそのラインに到達した瞬間に未来を予測して出るものではない。

ZigZagピボットと構造条件が揃った後に確定する。

---

### MTF特有の見え方がある

15分足MTAを1分足に表示しているため、15分足の情報が1分足上に展開される。

`lookahead_off` を使っているため先読みは避けているが、上位足が確定するタイミングと1分足上の見え方には差がある。

---

### 単一パターン依存

このstrategyは、15分MTA接触後の1分足トリガーのみを対象にしている。

相場局面によって成績が大きく変わる可能性がある。

## strategy v1.0 lite 検証メモ

### 結果

- 3年バックテストは問題なく動作
- 描画削除・アラート削除・チャート足MTA削除による軽量化は成功
- 複数パラメータを試したが、成績は不十分

### 暫定判断

- 15分MTA接触のみをセットアップとする現在の条件では優位性が弱い可能性
- 次はパラメータ最適化よりも、セットアップ品質向上とSL設計の見直しを優先する

### 次の改善候補

1. MTA守り足フィルター
2. MTA終値ブレイクでセットアップ無効
3. SL Mode追加
4. HTF EMAフィルター
5. 時間帯フィルター
