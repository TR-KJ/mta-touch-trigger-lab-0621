# 02_pine_rules.md

# Pine Script 開発ルール

このプロジェクトでは、PINE2026-05 のハマり集を前提に開発する。

特に、Pine Scriptでは `pine_check` / `pine_compile` が通っても、TradingView実機でランタイムエラーが出ることがあるため、実機確認を必須とする。

---

## 1. バージョン方針

本プロジェクトのベースコードは以下。

```pine
//@version=5
import DevLucem/ZigLib/1 as ZigZag
```

`DevLucem/ZigLib/1` はv5ライブラリのため、当面は `//@version=5` を維持する。

v6化は、ライブラリ依存を外すか、v6での挙動を十分確認してから行う。

---

## 2. 基本文法ルール

### 1行1文

Pine Scriptでは、セミコロンによる複数文記述をしない。

```pine
// NG
p1 = arr.get(n - 3); h1 = isH.get(n - 3)
```

```pine
// OK
p1 = arr.get(n - 3)
h1 = isH.get(n - 3)
```

---

## 3. 外スコープ変数の扱い

関数内から外スコープの変数を書き換える設計は避ける。

```pine
// NG
drawMta(price) =>
    lMta := line.new(...)
```

基本は、以下のどちらかにする。

```text
1. インラインで書く
2. UDTのフィールドとして管理する
```

本プロジェクトでは、まずは安全性優先でインライン実装を基本とする。

---

## 4. line / label の削除ルール

`line.delete()` / `label.delete()` 後は、必ず `:= na` で参照を無効化する。

```pine
if not na(l)
    line.delete(l)
    l := na
```

```pine
if not na(lb)
    label.delete(lb)
    lb := na
```

削除済みオブジェクトの参照を残すと、ダブル削除や想定外挙動の原因になる。

---

## 5. 水平線ルール

水平線を作る場合、`x1 == x2` にしない。

```pine
// NG
line.new(bar_index, price, bar_index, price, extend = extend.right)
```

```pine
// OK
line.new(bar_index, price, bar_index + 1, price, extend = extend.right)
```

`xloc.bar_time` を使う場合も、開始時刻と終了時刻を同一にしない。

---

## 6. ループルール

### `by -1` は使わない

Pineでは `for ... by -1` がランタイムエラーになることがある。

```pine
// NG
for i = n - 1 to 0 by -1
    ...
```

逆順にしたい場合は、原則として前方走査して最後にマッチしたindexを使う。

```pine
result = -1
if n > 0
    for i = 0 to n - 1
        if condition
            result := i
```

---

### `start > end` の逆走に注意

Pineの `for i = a to b` は、`a > b` のとき空ループではなく逆走する。

そのため、必ず `start <= end` を確認してからループする。

```pine
if start <= end
    for i = start to end
        ...
```

---

### 空配列ループ禁止

```pine
// NG
for i = 0 to array.size(a) - 1
    ...
```

```pine
// OK
if array.size(a) > 0
    for i = 0 to array.size(a) - 1
        ...
```

---

## 7. 条件式ルール

### `and` / `or` の短絡評価に頼らない

Pineでは、`and` / `or` の右辺が評価され、配列アクセスなどでランタイムエラーになることがある。

特に危険な例。

```pine
// NG
if array.size(a) == 0 or array.last(a) != x
    ...
```

安全な書き方。

```pine
addIt = false
if array.size(a) == 0
    addIt := true
else
    if array.last(a) != x
        addIt := true

if addIt
    ...
```

### 危険な右辺に置かないもの

以下は `and` / `or` の右辺に置かず、ネスト `if` でガードする。

- `array.get()`
- `array.last()`
- `map.get()`
- UDTフィールド参照
- `na` 前提の除算
- indexアクセス
- 外部ライブラリ由来の値

---

## 8. 配列ルール

### `.get()` / `.last()` 前にサイズ確認

```pine
if array.size(a) > 0
    v = array.last(a)
```

```pine
if idx >= 0
    if idx < array.size(a)
        v = array.get(a, idx)
```

---

### 並走配列は最小サイズを使う

複数配列を同じindexで扱う場合、必ず最小サイズを境界にする。

```pine
n1 = math.min(array.size(zP), array.size(zH))
n2 = math.min(array.size(zB), array.size(zT))
n = math.min(n1, n2)
```

以降のアクセスは `n` を基準にする。

---

### 保持indexは毎回境界確認

`var int idx` のようにバーをまたいでindexを保持する場合、アクセス前に必ず境界確認する。

```pine
validIdx = false
if idx >= 0
    if idx < array.size(a)
        validIdx := true

if validIdx
    v = array.get(a, idx)
else
    idx := -1
```

---

## 9. request.security ルール

`request.security()` では必ず以下を指定する。

```pine
lookahead = barmerge.lookahead_off
gaps = barmerge.gaps_off
```

例。

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

## 10. DevLucem/ZigLib 使用ルール

このプロジェクトでは、ベースコードで `DevLucem/ZigLib/1` を使う。

```pine
import DevLucem/ZigLib/1 as ZigZag
[direction, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
```

### UDT履歴参照に注意

ライブラリが返す `chart.point` などのUDTを直接履歴参照しない。

安全な方針。

```pine
zpx = z2.price
zix = z2.index
ztm = z2.time

pPrice = zpx[1]
pBar = na(zix[1]) ? -1 : int(zix[1])
pTime = na(ztm[1]) ? -1 : int(ztm[1])
```

UDTそのものではなく、float / int 系列に退避してから `[1]` 参照する。

---

## 11. MTA検出ロジック変更ルール

既存の `MTA v2.1` の検出ロジックは、原則として勝手に変更しない。

変更してよい場合。

```text
1. 明確なバグ修正
2. ユーザーが変更を依頼した場合
3. 別バージョンとして分岐する場合
```

v1では、MTA検出本体はなるべく保持し、その後ろにセットアップ/トリガー判定を追加する。

---

## 12. TradingView実機確認ルール

`pine_check` / `pine_compile` が通っても完了ではない。

必ずTradingViewチャートで確認する。

### 確認項目

- コンパイルエラーが出ない
- ランタイムエラーが出ない
- 1分足チャートで動作する
- 15分MTAが表示される
- SETUPラベルが接触位置で出る
- ENTRYラベルがトリガー位置で出る
- 古いスタディが残っていない

### 修正後の注意

TradingViewでは古いコンパイル済みスタディが残ることがある。

修正後は、以下を行う。

```text
1. チャート上の古いスタディを削除
2. マイスクリプトから再追加
3. 実機で再確認
```

---

## 13. パフォーマンスルール

### 配列の無制限成長に注意

ヒストリカルバーで配列が巨大化しすぎると重くなる。

必要に応じて、直近N件だけ保持する。

初期目安。

```text
200〜500件程度
```

ただし、v1ではまず挙動確認を優先し、必要になった時点でキャップを入れる。

---

## 14. このプロジェクトでの優先順位

優先順位は以下。

```text
1. ランタイムエラーを出さない
2. MTA検出ロジックを壊さない
3. 目視確認しやすくする
4. 後からstrategy化しやすくする
5. 複雑化しすぎない
```
