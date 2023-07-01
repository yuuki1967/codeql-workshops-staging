# CodeQL Workshop for JavaScript : tEnvoy の中の不適切な暗号電子署名の検証

- 分析対象の言語 : JavaScript
- 難しさレベル : 1/3

## Overview

 - [問題 説明](#problemstatement)
 - [ワークショップ](#workshop)
   - [セクション 1: Finding Sources](#section1)
   - [セクション 2: Finding Sinks](#section2)
   - [セクション 3: Data Flow](#section3)
   - [セクション 5: Final Query](#finalquery)
   - [Stretch Exercise](#stretchexercise)
 - [What's next](#whatsnext)
 
## 問題 説明  <a id="problemstatement"></a>

暗号化に関する欠陥問題は、2021年のOWASP トップ10の二番目に重要な問題です。検証における欠陥は、データの機密性、および整合性を侵害します。本ワークショップでは、[CVE-2021-32685](https://github.com/TogaTech/tEnvoy/security/advisories/GHSA-7r96-8g3x-g36m)に注目します。: [tEnvoy](https://tenvoy.js.org/) (`< v7.0.3`) の中の不適切な暗号電子署名の検証

このセキュリティ脆弱性は、フィールド名[`verified`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2150)で返された値を使って、[`verify()`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2138)が呼ばれた場合に起きます。 [`signed`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2138)パラメタ内に、SHA-512を使った署名が格納された場合、常に`true`を返してしまうからです。つまり、常に署名は正しいということです。次の関数[`verifyWithMessage()`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2169)は、メッセージ認証の検証をするために、`verify()`を使います。`verified`フィールドのチェックをする代わりに、必ず `true`を返す`verify()`を使います。このため、booleanとして返す値は、常に`true`となるために、適切に検証されません。

このセキュリティ脆弱性問題は、7.0.3よりも古いバージョンに存在します。修正版は:[patch commit](https://github.com/TogaTech/tEnvoy/commit/a121b34a45e289d775c62e58841522891dee686b)です。

CodeQLで、この脆弱性を検出するために、データフローのフレームワークを使います。このプログラムの中で、ソース（問題のあるデータの入力されたところ）とシンク(脆弱性の発生箇所)を認識して、ソース(source)からシンク(sink)間のフローを追跡するために、データフローライブラリを使います。

**Source**

In the `verify` function: 
  ``` javascript
     {
		verified: _nacl.sign.detached.verify(hash, signature, this.getPublic(_getPassword())),
		hash: signed.split("::")[0]
	 };
   ```
**Sink**

`verifyWithMessage` 関数:

`this.verify(signed, password)`

## ワークショップ  <a id="workshop"></a>

このワークショップでは、いくつかのステップに分割して説明します。それぞれのステップごとに１クエリを記述しまし、それを使って、改良していきます。

それぞれのステップごとに役立つCodeQLの標準ライブラリにあるclass、predicateのヒントが用意しています。VsCodeの中で、それぞれのclass,predicatesの自動補完機能、もしくは、jump-to-definition(CMD+right click on your mouse)コマンドで、説明や、コードの参照が可能です。

また、それに続いて、具体的な回答例を示すSolutionを見ることができます。ただし、これらは、クエリの抜粋です。クエリを描き始めるに際は、`import javascript`を記述することをお忘れなく。

### セクション 1: ソースを見つける  <a id="section1"></a>


1. 全ての関数を見つける
    <details>
    <summary>Hint</summary>

    CodeQL JavaScriptライブラリの`Function`を使用します。
    </details>
     <details>
    <summary>Solution</summary>
    
    ```CodeQL
    from Function f
    select f
    ```
    </details>
2. 全ての関数の中から、`verify`という名前の関数を見つけます。
    
    <details>
    <summary>Hint</summary>
    
     `Function.getName()`
  
    </details>

    <details>
  
    <summary>Solution</summary>
    
    ```CodeQL
  
    from Function f
    where f.getName() = "verify"
    select f
    
    ```
    (94 results)
    </details>


3. 前回のステップで、94個の"verify”関数を検出しました。ですが、これら全てが、脆弱性であるわけではありません。そのため、特定するための絞り込みが必要になります。`verified`属性を持つオブジェクトを含む関数のみに絞り込みましょう。CodeQLは、色々な JavaScriptの文法構成をサポートしています。[object expressions](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/Expr.qll/type.Expr$ObjectExpr.html)について、どう判断するのか、見つけるために、[documentation](https://codeql.github.com/docs/codeql-language-guides/codeql-library-for-javascript/)を参照できます。また、ドキュメントを表示するautocompletionを使って、知りたい情報を取得できます。さらに、`verified`属性と、プログラム構成をを理解するために、AST viewerを試すこともできます。

    <details>
  
    <summary>Hint</summary>
    
    ```CodeQL
  
      from Function f, ObjectExpr oe
      where f.getName() = "verify" and
            oe.getEnclosingFunction() = _ and  // TODO: replace `_` with the appropriate variable
            oe.getAProperty().getName() = _    // TODO: replace `_` with an appropriate string
      select oe
    
    ```
    </details>


    <details>
    <summary>Solution</summary>

      ```CodeQL
  
          from Function f, ObjectExpr oe
          where f.getName() = "verify" and
                oe.getEnclosingFunction() = f and
                oe.getAProperty().getName() = "verified"
          select oe
      ```

    (2 results)
    </details>

### セクション 2: シンクを見つける <a id="section2"></a>

1. 呼び出しもとは、オブジェクトがboolean値を使ったという事実から脆弱性問題が発覚したということを思い返してください。さらに特定すると、状態を持つ値として利用されています。CodeQLのJavaScriptライブラリは、状態として利用される表記を説明するclassを持ちます。: [ConditionGuardNode](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/CFG.qll/type.CFG$ConditionGuardNode.html)は、Control Flow Graphの中のnodeの型です。この型は、プログラムのある時点において、状態が真なのか偽なのかを記録します。`isCondition()`predicateを定義してみましょう。[exists](https://codeql.github.com/docs/ql-language-reference/formulas/#exists)を使います。

    <details>
  
    <summary>Hint</summary>
    
    ```CodeQL
       predicate isCondition(Expr expr) {
          // fill in the body of the predicate
          // Use Quick Evaluation to test the predicate
       }

      select 1  // stub query body
    
    ```
    </details>

    <details>
    <summary>Solution</summary>

      ```CodeQL

          predicate isCondition(Expr expr) { exists(ConditionGuardNode cgn | expr = cgn.getTest()) }

          select 1  // stub query body
      ```
  
    </details>

  
### セクション 3: データフロー <a id="section3"></a>

ここまでで、(a)`verified`属性を使ったオブジェクトを構成する関数、(b)状態が真か偽であるか指定している場所を認識できています。その次に、これら２つの条件を１つにすることを考えます。: 

プログラム分析において、_data flow_ 問題と呼んでいます。データフローは、"プログラムの中で、この記述は、かつて別の場所から生成した値を持ちますか？"というような質問に答えることをサポートします。

方向性のあるグラフパスを見つけるようなデータフロー問題を視覚化します。そのようなパスが存在した場合、データは、これら２つのノード間を流れます。
We can visualize the data flow problem as one of finding paths through a directed graph, where the nodes of the graph are elements in program, and the edges represent the flow of data between those elements. If a path exists, then the data flows between those two nodes.

JavaScript向けCodeQLは標準ライブラリでデータフロー分析を提供します。データフロー分析を実施する場合、クエリのトップで、`import semmle.code.javascript.dataflow.DataFlow`を挿入します。そのライブラリは、`DataFlow::Node`クラスを使って、ノードをモデル化しています。これらノードはASTノードとは分離されており、データフローを柔軟にモデル化できます。

いくつかデータフローノードの種類がここにあります。- 式ノード、パラメタノードが一般的です。

このセクションの中で、このテンプレートを追加することで、データフローのクエリを作成できます。


```CodeQL
/**
 * @kind path-problem
 */

import javascript
import DataFlow::PathGraph

class Config extends DataFlow::Configuration {
  Config() { this = "always true" }

  override predicate isSource(DataFlow::Node source) {
    // fill in the body of this member predicate
  }

  override predicate isSink(DataFlow::Node sink) {
    // fill in the body of this member predicate
    // you can use the isCondition() predicate we created earlier
  }
}

from Config config, DataFlow::PathNode source, DataFlow::PathNode sink
where            // fill in the where clause
select sink, source, sink, "always true"
```


1. 前回作成したクエリを使って`isSource`predicateを完成させます。 

    <details>
    <summary>Hint</summary>

    - query句からpredicateへ移行します: 
       - `from`で指定している宣言を`exists`の変数宣言へ移動
       - `exists`の本体に、`where`句の状態を置く 
       - predicateのパラメタの１つに`select`と同等にする状態を追加する 
   
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
      override predicate isSource(DataFlow::Node node) {
            exists(ObjectExpr obj |
               obj = source.asExpr() and
               obj.getAProperty().getName() = "verified"
            )
      }
     ```
    </details>

2. `isSink` predicate を完成させます
    <details>
    <summary>Hint</summary>
     上記と同様の処理を実装して完成させます。
   </details>
   <details>
    <summary>Solution</summary>

     ```ql
        override predicate isSink(DataFlow::Node sink) { isCondition(sink.asExpr()) }      
     ```
  </details>


これで本クエリを実行できます。結果を見てみましょう。

## クエリの完成形 <a id="finalquery"></a>
```CodeQL
/**
 * @kind path-problem
 */

import javascript
import DataFlow::PathGraph

predicate isCondition(Expr expr) { exists(ConditionGuardNode cgn | expr = cgn.getTest()) }

class Config extends DataFlow::Configuration {
  Config() { this = "always true" }

  override predicate isSource(DataFlow::Node source) {         
	  exists(ObjectExpr obj |
               obj = source.asExpr() and
               obj.getAProperty().getName() = "verified"
            )
  }

  override predicate isSink(DataFlow::Node sink) { isCondition(sink.asExpr()) }
}

from Config config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink, source, sink, "always true"
```
(2 results)


## What's next? <a id="whatsnext"></a>
- Read the [tutorial on analyzing data flow in JavaScript]().
- Go through more [CodeQL training materials for JavaScript]().
- Try out the latest CodeQL JavaScript Capture-the-Flag challenge on the [GitHub Security Lab website](https://securitylab.github.com/ctf) for a chance to win a prize! Or try one of the older Capture-the-Flag challenges to improve your CodeQL skills.
- Try out a CodeQL course on [GitHub Learning Lab](https://lab.github.com/githubtraining/codeql-u-boot-challenge-(cc++)).
- Read about more vulnerabilities found using CodeQL on the [GitHub Security Lab research blog](https://securitylab.github.com/research).
- Explore the [open-source CodeQL queries and libraries](https://github.com/github/codeql), and [learn how to contribute a new query](https://github.com/github/codeql/blob/master/CONTRIBUTING.md)

## Acknowledgements 

The GitHub Security Lab provided this query.
The initial version of this workshop was written by @zbazztian and developed further for presentation by @s-samadi

@rvermeulen , @hohn ,and @ps-apac provided valuable feedback. 
First version of this workshop was delivered by @s-samadi and @zbazztian