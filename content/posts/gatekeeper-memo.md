---
title: "Gatekeeper Memo"
date: 2020-12-13T15:47:15+09:00
draft: true
toc: true
tags:
  - k8s
  - rego
  - opa
  - gatekeeper
---

この記事は [FOLIO Advent Calendar 2020](https://adventar.org/calendars/5553) 20日目の記事です  

前回の記事でhelmを交えたargoCDの話をすると言ったな、あれは嘘だ  
調べてて面白そうだったので今回は [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) で利用されるpolicy記述言語Regoについてのおぼえ書きということにさせていただきます  

## Gatekeeper
policy記述言語Regoを利用してk8s上のマニフェストに対してpolicyを設定できるツール  
実装的にはk8s側ではAdmission Webhooks, Regoを利用するためにOPA, それぞれを利用して作られています  
これによってk8sマニフェストに対してpolicy制限を実現し、特定のキーの利用の強制や逆に制限をかけることができる  

---

今回はそのpolicyを記述するための言語Regoについて書きます  
実際にGatekeeperを触ってからの方がwhyを理解しやすいと思うのでREADMEの [How to Use Gatekeeper](https://github.com/open-policy-agent/gatekeeper#how-to-use-gatekeeper)を実施してから  
読むのを推奨します

## Rego
[policy記述言語](https://www.openpolicyagent.org/docs/latest/policy-language/)  
一般的な逐次処理を記載していくタイプの言語とは違い  
宣言的に条件を記載していき、結果値を得ることを目的としている  
prologとか定理証明系で使われている言語の流れを汲んでいるらしい(よく知らない)  

[playground](https://play.openpolicyagent.org/)も存在し、以降のコードスニペットはすべてplayground上で動くものを記載している  

### 変数  
```python
x := 3
```

regoの変数には利用方法が二種類ある  
ひとつは一般的なプログラミング言語同様値を束縛するためのもの  
もうひとつは少し曲者で未束縛の変数を利用した場合にその変数が取りうる可能性を持つ値の範囲で束縛する  
この場合「束縛」という表現は正しくないと思うが、自分的な理解しやすい表現なのであえてこう表現する  

```python
x := [1, 2, 3]

y {
  some i
  x[i]
  i == 0 # 1や2を指定してもtrue
}
```
上記の例では `i` は `x` のindexの範囲の値を取るため `i` は0もしくは1もしくは2となる  
ルール構文 `<ruleName> { .. }` に関しては今の所説明をしていないが後述する、一旦は一般的なプログラミング言語でいう  
ブロック構文に名前がついたものとでも思っていただければいい  
Regoでは代入式を除く殆どの式が値の絞り込みを行うためのもので、上記の例でもルール `y` の2行目 `x[i]` の時点では  
`i` は0~2を取りうる可能性があるが3行目 `i == 0` の時点で `i` は `0` で確定し、以降それ以外の値とはmatchしなくなる  

### Rule
Regoはpolicyを記述する言語でpolicyはいくつかのルールの集合として定義される  
ルールとは入力値を利用し、複数の式を適用した結果、取りうる残りの値を返す関数のようなものだ  

Regoではそれぞれの行の条件式は、それまでの行で検証された変数の取りうる値すべての範囲で検証され、trueの値の範囲のみ  
次の行を評価する  
すべてがfalseと評価された場合に取りうる結果がなくなったと判断され、未定義値を返す、これは戻り値のjsonとしては `undefined` として表現される  

```python
user_permission := {
  "bob": ["view"],
  "alice": ["view", "edit", "create"]
}

exists {
  user_permission[input.user_name]
}

permission[pm] {
  pm := user_permission[input.user_name][_]
  not pm == "view"
}
```

一つ目のルール `exists` は結果の取りうる値が一つの場合のルールだ  
inputは入力jsonの値を参照している、入力 `user_name` が `user_permission` に  
含まれる値であれば、結果値は一つの配列型の値として決定する  
結果値が一つの値に決定されている場合は、その値がbool型のfalseでない場合を除き  
全てtrueとして出力される  

二つ目のルール `permission` では変数 `pm` が取りうる値を全て返すルールだ  
`[_]` は未束縛の変数を利用する場合の簡略記法で、簡略化しない場合以下のものと同じだ
```python
permission[pm] {
  some i
  pm := user_permission[input.user_name][i]
```
明示的に変数指定をする場合は `i` を再利用できるが、今回は `i` には関心がないので簡略記法を使った  

変数 `pm` には変数 `i` が取りうる範囲で適用した場合の結果値が入る  
入力jsonを `{ "user_name": "alice" }` と想定した場合  
一行目の式では `i` には0~2の値が `pm` は `"view", "edit", "create"` の範囲で値を取ることがわかる  
二行目の式ではさらに `pm` の範囲を絞り込み `view` 以外の値をとることを示している  

つまり最終的な結果値 `pm` の取りうる範囲は `"edit", "create"` となり、出力jsonではこの配列を返す  
出力をobjectとすることで複数の値を含むこともできる  
```python
permission[{"kind": pm, index: i}] {
  some i
  pm := user_permission[input.user_name][i]
  not pm == "view"
}

# objectの配列を返すことができる
#"permission": [
#  {
#    "index": 1,
#    "kind": "edit"
#  },
#  {
#    "index": 2,
#    "kind": "create"
#   }
# ],
```

ルールはひとつのファイルに複数定義し、ルールから別のルールを参照することができるが  
Gatekeeperを利用する場合は `violation` ルール名で定義したものがエントリーポイント的に扱われる  

RegoでのPolicyの定義は複数のルールによって構成されるが、結果的に出力値がallowなのかdenyなのかということはRego側では関心を持たない  
つまり、`undefined` が出力されようが `false` が出力されようがRegoにとってそれはあくまでただの出力値であり  
それをallowなのかdenyなのかを判断するのは利用側(多くの場合はOPA)ということになる  

### some Keyword
`some` は未束縛の変数を定義するキーワードだ  
Regoにはスコープの概念が存在し、内側のスコープから外側の変数を参照することができる  
Regoでは見束縛の変数であることが非常に重要なため、誤って外側の変数を参照しないよう、`some` キーワードで明示的に宣言することができる  
```python
i := 1
x := [1, 2, 3]

y {
  some i # この行をコメントアウトした場合 i == 1 に決定されているため、3行目でfalseとなる
  x[i]
  i == 0 # 1や2を指定してもtrue
}
```

### 内包表記
```python
x := [1, 2, 3]

# array comprehensions
xx := [inc | inc := x[_] + 1]

# object comprehensions
xbj := {k: v|
          some i
          k := x[i]
          v := x[i] + 1
       }

# set comprehensions
xset := {inc | inc := x[_] + 1}
```
配列やobject, setには内包表記が存在する  
内包表記自体は一般的なプログラミング言語と同じ動作をするが、式部分はルール同様の式が記載できる  
また、setとobjectの内包表記がどちらも `{..}` で、混同しないように注意  
これはsetのリテラルが `{ 1, 2, 3 }` といった形になっているため ~~ややこしい~~  

---
Gatekeeperを利用する場合でのRegoはこれらの構文だけ覚えておけば大丈夫かもしれない
