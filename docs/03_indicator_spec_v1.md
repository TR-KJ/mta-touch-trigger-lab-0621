# 03_indicator_spec_v1.md

# インジケーター版 v1 仕様書

---

## 1. バージョン名

```text
MTA Touch Trigger Indicator v1
```

略称。

```text
MTT v1
```

---

## 2. 開発目的

15分足MTAへの接触をセットアップとし、1分足でトリガー条件を満たした場合にエントリー候補を表示するインジケーターを作成する。

v1では、バックテスト用のstrategyではなく、目視確認用のindicatorとして作成する。

---

## 3. ベースコード

ベースコードは以下。

```text
MTA v2.1
```

主な仕様。

```text
//@version=5
import DevLucem/ZigLib/1 as ZigZag
```

15分MTA取得には、既存コードの選択足MTA取得ロジックを利用する。

```pine
[sM, sTr, sMt, sEnd] = request.security(
     syminfo.tickerid,
     selTF,
     f_mtaCalc(),
     lookahead = barmerge.lookahead_off,
     gaps = barmerge.gaps_off
)
```

---

## 4. 想定チャート

v1の想定は以下。

```text
チャート足：1分足
選択足TF：15分
```

初期値として、選択足TFは `15` を推奨する。

```pine
selTFin = input.timeframe('15', '選択足TF')
```

ただし、検証用に他の時間足へ変更できるようにする。

---

## 5. 使用する値

既存の `MTA v2.1` から、主に以下を使用する。

| 変数 | 意味 |
|---|---|
| `sM` | 選択足MTA価格 |
| `sTr` | 選択足MTA方向 |
| `sMt` | 選択足MTA発生時刻 |
| `sEnd` | 選択足MTA終了済み状態 |

---

## 6. MTA方向の扱い

### 上昇MTA

```pine
sTr == 1
```

ロング候補の上位足MTAとして扱う。

---

### 下降MTA

```pine
sTr == -1
```

ショート候補の上位足MTAとして扱う。

---

### 終了済みMTA

```pine
sTr == 0
```

v1では新規セットアップ対象外とする。

---

## 7. セットアップ条件

### MTA接触判定

1分足のローソク足が15分MTA価格に接触した場合、セットアップ発生とする。

```pine
mtaTouched = low <= sM and high >= sM
```

ただし、実装時は `and` の短絡評価に頼らず、ネスト `if` で安全に判定する。

---

### Long Setup

以下をすべて満たす場合、Long Setupとする。

```text
1. sTr == 1
2. sM が na ではない
3. sEnd ではない
4. 1分足の low <= sM
5. 1分足の high >= sM
```

意味。

```text
15分足が上昇MTA状態
かつ
1分足が15分MTAに接触
```

---

### Short Setup

以下をすべて満たす場合、Short Setupとする。

```text
1. sTr == -1
2. sM が na ではない
3. sEnd ではない
4. 1分足の low <= sM
5. 1分足の high >= sM
```

意味。

```text
15分足が下降MTA状態
かつ
1分足が15分MTAに接触
```

---

## 8. セットアップ発生時に保存する値

Long Setup または Short Setup が発生したら、以下を保存する。

| 保存値 | 内容 |
|---|---|
| `setupDir` | 方向。Longは `1`、Shortは `-1` |
| `setupBar` | セットアップ発生バー |
| `setupMta` | 接触した15分MTA価格 |
| `setupHigh` | セットアップ足の高値 |
| `setupLow` | セットアップ足の安値 |
| `setupTime` | セットアップ発生時刻 |

---

## 9. トリガー条件

v1では、接触足の高値/安値ブレイクをトリガーとする。

---

### Long Trigger

Long Setup発生後、一定本数以内に、終値がセットアップ足高値を上抜けたらLong Trigger成立。

```text
close > setupHigh
```

この時、ENTRY候補を表示する。

---

### Short Trigger

Short Setup発生後、一定本数以内に、終値がセットアップ足安値を下抜けたらShort Trigger成立。

```text
close < setupLow
```

この時、ENTRY候補を表示する。

---

## 10. 有効期限

セットアップ発生後、指定本数以内にトリガーが成立しなければセットアップ失効とする。

初期値。

```text
60本
```

1分足チャートでは60分相当。

入力項目として変更可能にする。

```pine
expireBars = input.int(60, 'セットアップ有効本数', minval = 1)
```

---

## 11. セットアップ状態

v1では、同時に保持するセットアップは1つとする。

### 状態変数

| 変数 | 内容 |
|---|---|
| `waitingTrigger` | トリガー待ち状態 |
| `setupDir` | セットアップ方向 |
| `setupBar` | セットアップ発生バー |
| `setupMta` | セットアップMTA価格 |
| `setupHigh` | セットアップ足高値 |
| `setupLow` | セットアップ足安値 |

---

## 12. セットアップ更新ルール

### 新規セットアップ

トリガー待ち状態ではない場合、MTA接触で新規セットアップを作る。

```text
waitingTrigger == false
かつ
MTA接触
```

---

### トリガー待ち中

トリガー待ち中は、原則として新しいセットアップで上書きしない。

理由。

```text
v1では挙動を単純にするため
```

---

### 失効

以下の場合、セットアップを解除する。

```text
bar_index - setupBar > expireBars
```

---

### トリガー成立

トリガー成立時、ENTRY表示後にセットアップを解除する。

---

### MTA方向変化

トリガー待ち中に `sTr` が `0` または逆方向になった場合、セットアップを解除する。

例。

```text
Long Setup待ち中に sTr != 1
Short Setup待ち中に sTr != -1
```

---

## 13. 表示仕様

### MTAライン

既存の選択足MTAライン表示を利用する。

初期表示。

```text
15分MTAライン：表示
チャート足MTAライン：任意
```

---

### SETUPラベル

セットアップ発生バーに表示する。

#### Long Setup

```text
SETUP L
```

表示位置。

```text
low付近
```

#### Short Setup

```text
SETUP S
```

表示位置。

```text
high付近
```

---

### ENTRYラベル

トリガー成立バーに表示する。

#### Long Entry

```text
ENTRY L
```

表示位置。

```text
low付近
```

#### Short Entry

```text
ENTRY S
```

表示位置。

```text
high付近
```

---

### トリガーライン

セットアップ足の高値/安値にトリガーラインを表示する。

#### Long

```text
setupHigh に水平ライン
```

#### Short

```text
setupLow に水平ライン
```

ラインは `x1 == x2` にならないようにする。

---

## 14. アラート仕様

v1では、以下のアラートを追加候補とする。

### Setup Alert

```text
Long Setup 発生
Short Setup 発生
```

### Entry Alert

```text
Long Entry 候補
Short Entry 候補
```

初期v1では、ラベル表示を優先し、アラートは後から追加してもよい。

---

## 15. 入力項目

v1で追加する入力項目。

| 入力名 | 初期値 | 内容 |
|---|---:|---|
| `enableTouchSetup` | true | MTA接触セットアップを有効化 |
| `expireBars` | 60 | セットアップ有効本数 |
| `showSetupLabel` | true | SETUPラベル表示 |
| `showEntryLabel` | true | ENTRYラベル表示 |
| `showTriggerLine` | true | トリガーライン表示 |

---

## 16. v1では実装しないもの

v1では、以下は実装しない。

```text
strategy.entry
strategy.exit
SL/TP自動計算
RR指定
時間帯フィルター
曜日フィルター
EMAフィルター
複数セットアップ同時保持
複数ポジション管理
勝率/PF計算
```

---

## 17. 開発時の注意点

### MTA検出ロジックを壊さない

`f_mtaCalc()` は原則そのまま使用する。

変更する場合は、明確な理由を記録する。

---

### DevLucem/ZigLibの扱い

`DevLucem/ZigLib/1` を使うため、v5を維持する。

```pine
//@version=5
```

---

### 配列サイズ

`zP / zH / zB / zT` の最小サイズを `n` にする。

```pine
n1 = math.min(array.size(zP), array.size(zH))
n2 = math.min(array.size(zB), array.size(zT))
n = math.min(n1, n2)
```

---

### 実機確認

コード作成後は必ずTradingViewで確認する。

```text
1. Pineエディタに貼り付け
2. 保存
3. チャートに追加
4. エラー確認
5. 修正後は古いスタディを削除
6. 再追加
```

---

## 18. v1完了条件

v1の完了条件は以下。

```text
1. コンパイルエラーがない
2. TradingView実機でランタイムエラーが出ない
3. 1分足チャートで15分MTAが表示される
4. 15分MTA接触でSETUPラベルが出る
5. 接触足高値/安値ブレイクでENTRYラベルが出る
6. 有効期限切れでセットアップが解除される
7. MTA方向変化時にセットアップが解除される
```

---

## 19. 次バージョン候補

v1確認後、以下を検討する。

### v1.1

```text
SETUP/ENTRYアラート追加
```

### v1.2

```text
EMA20回復トリガー追加
```

### v1.3

```text
直近1分ピボット高値/安値ブレイク追加
```

### v2.0

```text
strategy版作成
SL/TP/RR検証
```

# docs/03_indicator_spec_v1.md 追記分

---

## 20. v1.1 追加仕様：Touch Trigger専用アラート

v1.1では、既存のMTA発生/終了アラートとは別に、Touch Trigger専用アラートを追加した。

### 追加したアラート条件

```text
TT SETUP L
TT SETUP S
TT ENTRY L
TT ENTRY S
```

### 目的

MTAそのものの発生/終了ではなく、手法上のセットアップおよびエントリー候補を個別に通知できるようにする。

---

### SETUPアラート

#### TT SETUP L

Long Setup発生時に通知する。

条件。

```text
15分MTAが上昇状態
かつ
1分足が15分MTAに接触
```

#### TT SETUP S

Short Setup発生時に通知する。

条件。

```text
15分MTAが下降状態
かつ
1分足が15分MTAに接触
```

---

### ENTRYアラート

#### TT ENTRY L

Long Entry候補発生時に通知する。

条件。

```text
Long Setup発生後、
選択中のトリガー方式でLong Entry条件が成立
```

#### TT ENTRY S

Short Entry候補発生時に通知する。

条件。

```text
Short Setup発生後、
選択中のトリガー方式でShort Entry条件が成立
```

---

### alertcondition

TradingViewのアラート作成画面で、以下の条件を個別に選択できる。

```pine
alertcondition(longSetupSignal, title = 'TT SETUP L', message = '{{ticker}} Touch Trigger SETUP L')
alertcondition(shortSetupSignal, title = 'TT SETUP S', message = '{{ticker}} Touch Trigger SETUP S')
alertcondition(longEntrySignal, title = 'TT ENTRY L', message = '{{ticker}} Touch Trigger ENTRY L')
alertcondition(shortEntrySignal, title = 'TT ENTRY S', message = '{{ticker}} Touch Trigger ENTRY S')
```

---

## 21. v1.2 追加仕様：トリガー方式選択

v1.2では、ENTRYトリガー方式を選択式にした。

### 追加したトリガー方式

```text
Setup Candle Break
EMA Reclaim
LTF Pivot Break
```

---

### 21.1 Setup Candle Break

v1.0からの標準トリガー。

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

### 役割

最もシンプルな価格ブレイク系トリガー。

他トリガー方式と比較するための基準条件として扱う。

---

### 21.2 EMA Reclaim

選択EMAを使った回復/割れトリガー。

#### 追加入力

```pine
emaLen = input.int(20, 'EMA期間', minval = 1, group = 'EMA Reclaim')
showEma = input.bool(true, 'EMA表示', group = 'EMA Reclaim')
```

#### Long

```text
Long Setup発生後、
1分足終値がEMAを下から上へ回復
```

条件。

```text
close > EMA
かつ
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
かつ
close[1] >= EMA[1]
```

### 役割

MTA接触後に、短期的な戻り・反転の勢いが出たかを確認するトリガー。

EMA期間は検証用に変更可能。

---

### 21.3 LTF Pivot Break

1分足の直近ピボット高値/安値を使った構造系トリガー。

#### 追加入力

```pine
pivotLeft = input.int(3, 'Pivot Left', minval = 1, group = 'LTF Pivot Break')
pivotRight = input.int(3, 'Pivot Right', minval = 1, group = 'LTF Pivot Break')
```

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

### 役割

単純なローソク足ブレイクではなく、下位足の小さな構造転換をトリガーとして扱う。

---

## 22. v1.2 実機確認結果

v1.2では以下をTradingView実機で確認済み。

```text
1. Setup Candle Break でv1.1同様に動作
2. EMA Reclaim でEMA線が表示される
3. EMA期間変更によりENTRY位置が変化する
4. LTF Pivot Break でPivot L/R変更によりENTRY位置が変化する
5. TT SETUP / TT ENTRY アラート条件がTradingView上に表示される
```

### 判定

```text
v1.2 仮合格
```

---

## 23. strategy版への移行方針

v1.2のインジケーター仕様をベースに、次段階としてstrategy版を作成する。

strategy版では、以下を追加する。

```text
strategy.entry
strategy.exit
SL設定
TP設定
RR設定
2%リスクベースの数量計算
手数料・スリッページ設定
```

ただし、MTA検出ロジック、SETUP判定、トリガー判定の基本構造は変更しない。
