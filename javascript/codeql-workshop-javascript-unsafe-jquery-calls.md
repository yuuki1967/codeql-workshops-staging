# CodeQL workshop for JavaScript: jQuery `$` 関数への危険な呼び出しを見つける

- 分析対象言語 : JavaScript
- 難しさレベル : 1/3

## 問題の説明 

jQueryは、非常に人気のあるフロントエンドのライブラリですが、古い、オープンソースライブラリで、HTMLドキュメントの計算や、操作、イベントハンドリング、アニメーション、Ajaxのような簡単なことするために設計されてたものです。jQueryは、機能拡張するためのプラグインをサポートします。Bootstrapもまた、JavaScriptにおいて人気のあるライブラリで、jQueryのプラグイン機能を利用しています。しかし、Bootstrap内部の、jQueryプラグインは、クロスサイトスクリプティング(XSS)攻撃の脆弱性のあるコードで実装されています。XSSとは、攻撃者が、Webアプリケーションに悪意あるコード(多くの場合、ブラウザのスクリプト)を別のユーザに送りつけることです。

Bootstrap jQuery プラグインの関連する４つの脆弱性は、[this pull request](https://github.com/twbs/bootstrap/pull/27047)で修正済みで、 それぞれの脆弱性は、CVEに登録されました。

これらプラグインの間違いの根幹は、プラグインに渡されるオプションを処理するjQeury `$`関数の利用でした。例えば、単純な以下のjQueryプラグイン:

```javascript
let text = $(options.textSrcSelector).text();
```

このプラグインは、CSSセレクタとして、`options.textSrcSelector`を評価して、どのHTML要素からテキストを読み込むかを決定すル、もしくは意図的に指定します。この例の中での問題は、`$(options.textSrcSelector)`が`options.textSrcSelector`に入る値が、`"<img src=x onerror=alert(1)>".`のような文字列の場合、Javascriptとしてコードを実行してしまうことです。

セキュリティ用語において、jQueryプラグインオプションがユーザ入力の**source**となり、`$`の引数がXSSが実行される**sink**となります。
上記リンクのpull requestはより安全にするための１つのアプローチを示します。:より特別化した、安全な関数`$`の代わりに`$(document).find`を使います。
```javascript
let text = $(document).find(options.textSrcSelector).text();
```

このセッションでは、Bootstrapのソースコードを解析するCodeQLを使います。ただし、利用するコードは、これら脆弱性問題を解決する前のものを使い、脆弱性を見つけるものです。

## セットアップ手順 

### ローカルPC上でクエリを記述します(recommended)

CodeQLクエリをBootstrapにおいて実行するために、次の手順に従い進めます。:

1. Visual Studio Code IDEのインストール
2. [CodeQL extension for Visual Studio Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)のダウンロード＆インストール。 完全なセットアップ手順は、[here](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)を参照下さい。
3. [Set up the starter workspace](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/#starter-workspace)をセットアップします。
    - **Important**: 標準のクエリライブラリもクローンするために、ローカルにクローンする際は、`git clone --recursive` もしくは `git submodule update --init --remote`を指定することを忘れないように。 
4. VSCodeを実行して、次のようにワークスペースをオープンします。: File > Open Workspace > `vscode-codeql-starter/vscode-codeql-starter.code-workspace`をブラウズします。
5. データベースを解凍します。
6. データベースをCodeQLデータベースとしてマウントします。
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the + sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.
7. `UnsafeDollarCall.ql`ファイルを `codeql-js-goof-workshop`ディレクトリに作成します。

## 参考資料
以下のリンク先も参考になります。
- [Learning CodeQL](https://codeql.github.com/docs/codeql-overview/)
- [Learning CodeQL for JavaScript](https://codeql.github.com/docs/codeql-language-guides/codeql-for-javascript/)
- [Using the CodeQL extension for VS Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)

## クエリの作成

XSSの脆弱性を見つけるクエリを作成するために、いくつかのステップに分離します。ステップごとにクエリを作成して、動作を確認します。

それぞれのステップには、役にたつclass、predicateの**Hint**として記載します。これらは CodeQLのJavaScript用標準ライブラリとなります。VSCodeでは、自動補完機能、もしくは、jump-to-definition(CMD+right click on mouse)でライブラリ情報を取得できます。

それから、それぞれのステップで、具体的実装例を紹介しています。**Solution**を展開すると参照できます。クエリは一部です。書き始める際は、`import javascript`から始める必要があります。

### jQuery `$` 関数を検出する

1. 全ての関数コールの記述を見つける
    <details>
    <summary>Hint</summary>

    関数コールを見つけるためには、CodeQL JavaScript ライブラリの`CallExpr`を使います。
    </details>
     <details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall
    select dollarCall
    ```
    </details>

2. それぞれの関数コールの第１引数を見つける
    <details>
    <summary>Hint</summary>

    `Expr`, `CallExpr.getArgument(int)`, `and`, `where`
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall, Expr dollarArg
    where dollarArg = dollarCall.getArgument(0)
    select dollarArg
    ```
    </details>

1. それら関数コールの中から`$`関数のみを検出する
    <details>
    <summary>Hint</summary>

    `CallExpr.getCalleeName()`
    </details><details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall, Expr dollarArg
    where
      dollarArg = dollarCall.getArgument(0) and
      dollarCall.getCalleeName() = "$"
    select dollarArg
    ```
    </details>

### jQuery plugin options(プラグインオプション)へのアクセスを見つける
次のセクションになります。新しくクエリファイルを作成するか、前回のクエリファイルを利用します。前回と同じクエリファイルを利用する際は、前回実装したsolution(ソリューション)は、コメントアウト(//, /* */)してください。

1. jQuery plugin optionへのアクセスがあった場合、 `something.options.optionName`を使って、それを判定します。 まず、property name(属性名)`options`を探します
    <details>
    <summary>Hint</summary>

    属性へのアクセスは、 CodeQL JavaScript ライブラリの`PropAccess`が対応します。属性を見つけるためには、`PropAccess.getPropertyName()`を使います。
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from PropAccess optionsAccess
    where optionsAccess.getPropertyName() = "options"
    select optionsAccess
    ```
    </details>

1. 次に先で実装したステップに対して、`something.options.optionName`の属性へのアクセスのベースを見つけるよう改良します
    <details>
    <summary>Hint</summary>

    ここに２つの属性アクセスがあります。２つ目は、１つ目の結果から属性アクセスを探すように記述します。`PropAccess.getBase()`は、どの属性がアクセスされるのかといったオブジェクトを見つけることができます。
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from PropAccess optionsAccess, PropAccess nestedOptionAccess
    where
      optionsAccess.getPropertyName() = "options" and
      nestedOptionAccess.getBase() = optionsAccess
    select nestedOptionAccess
    ```
    </details>

### これまでのステップを１つに統合します

1. これまでセクションで実装したクエリを１つに統合します。jQuery `$`関数の引数として利用される`something.options.optionName`フォームの連結している属性アクセスを見つけます。
    <details>
    <summary>Hint</summary>
    `from`セクションの中に必要とする全ての変数を定義します。そして、全ての論理状態を統合するために`and`論理演算子を使います。
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall, Expr dollarArg, PropAccess optionsAccess, PropAccess nestedOptionAccess
    where
      dollarCall.getArgument(0) = dollarArg and
      dollarCall.getCalleeName() = "$" and
      optionsAccess.getPropertyName() = "options" and
      nestedOptionAccess.getBase() = optionsAccess and
      dollarArg = nestedOptionAccess
    select dollarArg
    ```
    </details>

1. (おまけ) このステップ２へのソリューションは、結果として、まだパッチの当たっていないBootstrapに存在する３つの問題を検出するクエリです。それらのうちの２つは、この問題を対策したpull requestで修正された、適切な検出です。追加の問題については、このクエリでは、発見できません。例えば、jQuery option (`something.options.optionName`)は、`$`への引数として、常に使用するわけではなく、一旦ローカル変数に渡されるかもしれません。

    中間変数を利用や、ネストされた式（記述）は、検出された**データフロー分析**を必要とする典型的なソースコードの例です。 
    
    この脆弱性の異形を見つけるためには、少々JavaScript データフローライブラリを使う際にクエリを調整してみます。

    <details>
    <summary>Hint</summary>

    -  もし、`Expr`のようなASTノードがある場合、[`flow()`](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/AST.qll/predicate.AST$AST$ValueNode$flow.0.html)は、この表現（記述）へ/からフロー情報を推論する__data flow node__に変換します。

    - データフローノードがある場合、[`getALocalSource()`](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/dataflow/DataFlow.qll/predicate.DataFlow$DataFlow$Node$getALocalSource.0.html)が同一関数の中で、最終的にこのノードで完了する別のデータフローを検出します。 
    - もし、データフローノードがある場合、`asExpr()`は、AST記述中にそのデータフローノードを戻します。 
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall, Expr dollarArg, PropAccess optionsAccess, PropAccess nestedOptionAccess
    where
      dollarCall.getArgument(0) = dollarArg and
      dollarCall.getCalleeName() = "$" and
      optionsAccess.getPropertyName() = "options" and
      nestedOptionAccess.getBase() = optionsAccess and
      dollarArg.flow().getALocalSource().asExpr() = nestedOptionAccess
    select dollarArg, nestedOptionAccess
    ```
    </details>

## acknowledgement

This is a reduced version of a Capture-the-Flag challenge devised by @esbena, available at https://securitylab.github.com/ctf/jquery. Try out the full version!
