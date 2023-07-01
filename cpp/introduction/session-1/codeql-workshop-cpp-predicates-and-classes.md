# CodeQL workshop for C/C++: Predicates と classes

このワークショップの次のパートで、ここでのクエリをCodeQLの別の特徴を使って、リファクタリングします。ここでは、エクササイズはありません。

## Existential quantifiers

最初の探索する機能は、existential quantifiersです。この言葉はロジック、ロジックプログラム独特なものです。*existential qualifiers*をわかりやすく説明すると、関連した状態を持った一時的な変数のことです。文法は：
```
exists(<variable declarations> | <formula>)
```
existsは`from`と`where`句と似た構造を持ちます。最初は、１つ以上のせんげん、２つ目は、それらの変数を適用した式("conditions")を指定します。

これを使ってこのクエリをリファクタすることができます。
```ql
from IfStmt ifStmt
where
  exists(Block block |
    ifStmt.getThen() = block and
    block.getNumStmt() = 0
  )
select ifStmt, "Empty if statement"
```

## Predicates

その他の特徴として、_predicates_があります。再利用するためにプログラムの中にロジックの一部を隠蔽する(抽象化する)ための方法を提供します。一番シンプルな`from`-`where`-`select`で、`select`に結果テーブルの中の行もしくは、タプル（複数の要素構成）も出力することができます。

空のブロックを抽出するクエリの中に新しいpredicateを紹介します。

```ql
predicate isEmptyBlock(Block block) {
  block.getNumStmt() = 0
}

from IfStmt ifStmt
where isEmptyBlock(ifStmt.getThen())
select ifStmt, "Empty if statement"
```

結果の型を使って、predeicateで置き換えることで、結果を持ったpredicateを定義することが可能です。これは、特別な変数`result`を紹介します。

```ql
Block getAnEmptyBlock() {
  result.getNumStmt() = 0
}
```
実装の観点で、前回のpredicateと同じ結果を得ることできます。
```ql
predicate getAnEmptyBlock(Block result) {
  result.getNumStmt() = 0
}
```
主な違いは、使い方になります：
```ql
from IfStmt ifStmt
where ifStmt.getThen() = getAnEmptyBlock()
select ifStmt, "Empty if statement"
```
このpredicateは、式で、等価比較で利用することができます。しかしながら、predicateの両方のフォームは、内部で同一の値を計算します。

## Classes

このワークショップの最後のパートで、CodeQLのclassについて説明します。 classsはCodeQLの中で新しい型を定義できます。こちらも、再利用、構造化コードをする際の簡単な手法を提供するものです。

CodeQLの中ですべての型のように、classは、いくつかの値の一まとまりにしたものです。例えば、`BlockStmt`型は、classで、プログラムの中で、すべてのブロックのセットを表現しています。そのclassのセットを指定するロジカルな状態のセットを定義すると同様と考えられます。

例えば、空のブロックを表現するために新たにCodeQLのclassを定義できます。:
```ql
class EmptyBlock extends BlockStmt {
  EmptyBlock() {
    this.getNumStmt() = 0
  }
}
```
新たにclassを作成するとき、`class`を使って、`super-type`を提供します。QLの中ですべてのclassは最低でも`super-type`を持ち、初期値を持った`super-type`である必要があります。この例では、`EmptyBlock`は、`BlockStmt`classのすべての値を持って開始します。しかし、もう一方のclassと同じ値だけを持つclassだったら、特に別classにするメリットはありません。そのため、さらに値を限定を追加する状態定義するpredicateを実装します。この例では、`BlockStmt`classの性質にどれが`getNumStmt() = 0`を条件に当てはまるのか制限するものを追加しています。ここで、変数`this`は、生成した`BlockStmt`のインスタンスを参照を示し、利用することができます。

１つの値は、これらのセットの１つ以上含めることができます。つまり。１つ以上の型を持てます。例えば、空のブロックは、`BlockStmt`と`EmptyBlock`のどちらでもあることを意味します。

このclassは、実際に上述したとおり。predicateのソリューションと等価です。事実、同じ状態で、同じ値の計算結果となる。
```ql
from IfStmt ifStmt, EmptyBlock block
where ifStmt.getThen() = block
select ifStmt, "Empty if statement"
```
This is another instance of the proscriptive typing of QL - by changing the type of the variable to `EmptyBlock`, we change the meaning of the program.

As discussed previously, classes can also provide operations. These operations are called _member predicates_, as they are predicates which are members of the class. For example:
```ql
class MyBlock extends BlockStmt {
  predicate isEmptyBlock() {
    this.getNumStmt() = 0
  }
}
```
In this case we are not going to provide a characteristic predicate - this class is going to represent the same set of values as `Block`. However, we will provide a member predicate is specify whether this is an empty block. Member predicates also have a special variable called `this` which refers to the instance.

We can then use this in the same way as operations provided on standard library classes:
```ql
from IfStmt ifStmt, MyBlock block
where
  ifStmt.getThen() = block and
  block.isEmptyBlock()
select ifStmt, "Empty if statement"
```
In fact, this is how the standard library classes are implemented - if we select `getThen()`, we can see it is also defined as a member predicate.
