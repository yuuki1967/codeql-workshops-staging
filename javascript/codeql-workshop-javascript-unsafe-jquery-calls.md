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
5. Create an account on LGTM.com if you haven't already. You can log in via OAuth using your Google or GitHub account.
1. Visit the [database downloads page for the vulnerable version of Bootstrap on LGTM.com](https://lgtm.com/projects/g/esbena/bootstrap-pre-27047/ci/#ql).
1. Download the latest database for JavaScript.
1. Unzip the database.
1. Import the unzipped database into Visual Studio Code:
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the + sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.
1. Create a new file, name it `UnsafeDollarCall.ql`, save it under `codeql-custom-queries-javascript`.

### Writing queries in the browser

To run CodeQL queries on Bootstrap online, follow these steps:

1. Create an account on LGTM.com if you haven't already. You can log in via OAuth using your Google or GitHub account.
1. [Start querying the Bootstrap project](https://lgtm.com/query/project:1510734246425/lang:javascript/).
    - Alternative: Visit the [vulnerable Bootstrap project page](https://lgtm.com/projects/g/esbena/bootstrap-pre-27047) and click **Query this project**.


## Documentation links
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://codeql.github.com/docs/codeql-overview/)
- [Learning CodeQL for JavaScript](https://codeql.github.com/docs/codeql-language-guides/codeql-for-javascript/)
- [Using the CodeQL extension for VS Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)

## Challenge
The challenge is split into several steps. You can write one query per step, or work with a single query that you refine at each step.

Each step has a **Hint** that describe useful classes and predicates in the CodeQL standard libraries for JavaScript and keywords in CodeQL. You can explore these in your IDE using the autocomplete suggestions and jump-to-definition command.

Each step has a **Solution** that indicates one possible answer. Note that all queries will need to begin with `import javascript`, but for simplicity this may be omitted below.

### Finding calls to the jQuery `$` function

1. Find all function call expressions.
    <details>
    <summary>Hint</summary>

    A function call is called a `CallExpr` in the CodeQL JavaScript library.
    </details>
     <details>
    <summary>Solution</summary>
    
    ```
    from CallExpr dollarCall
    select dollarCall
    ```
    </details>

1. Identify the expression that is used as the first argument for each call.
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

1. Filter your results to only those calls to a function named `$`.
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

### Finding accesses to jQuery plugin options
Consider creating a new query for these next few steps, or commenting out your earlier solutions and using the same file. We will use the earlier solutions again in the next section.

1. When a jQuery plugin option is accessed, the code generally looks like `something.options.optionName`. First, identify all accesses to a property named `options`.
    <details>
    <summary>Hint</summary>

    Property accesses are called `PropAccess` in the CodeQL JavaScript libraries. Use `PropAccess.getPropertyName()` to identify the property.
    </details>
    <details>
    <summary>Solution</summary>
    
    ```
    from PropAccess optionsAccess
    where optionsAccess.getPropertyName() = "options"
    select optionsAccess
    ```
    </details>

1. Take your query from the previous step, and modify it to find chained property accesses of the form `something.options.optionName`.
    <details>
    <summary>Hint</summary>

    There are two property accesses here, with the second being made upon the result of the first. `PropAccess.getBase()` gives the object whose property is being accessed.
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

### Putting it all together

1. Combine your queries from the two previous sections. Find chained property accesses of the form `something.options.optionName` that are used as the argument of calls to the jQuery `$` function.
    <details>
    <summary>Hint</summary>
    Declare all the variables you need in the `from` section, and use the `and` keyword to combine all your logical conditions.
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

1. (Bonus) The solution to step 2 should result in a query with three alerts on the unpatched Bootstrap codebase, two of which are true positives that were fixed in the linked pull request. There are however additional vulnerabilities that are beyond the capabilities of a purely syntactic query such as the one we have written. For example, the access to the jQuery option (`something.options.optionName`) is not always used directly as the argument of the call to `$`: it might be assigned first to a local variable, which is then passed to `$`.

    The use of intermediate variables and nested expressions are typical source code examples that require use of **data flow analysis** to detect.

    To find one more variant of this vulnerability, try adjusting the query to use the JavaScript data flow library a tiny bit, instead of relying purely on the syntactic structure of the vulnerability. See the hint for more details.

    <details>
    <summary>Hint</summary>

    - If we have an AST node, such as an `Expr`, then [`flow()`](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/AST.qll/predicate.AST$AST$ValueNode$flow.0.html) will convert it into a __data flow node__, which we can use to reason about the flow of information to/from this expression.
    - If we have a data flow node, then [`getALocalSource()`](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/dataflow/DataFlow.qll/predicate.DataFlow$DataFlow$Node$getALocalSource.0.html) will give us another data flow node in the same function whose value ends up in this node.
    - If we have a data flow node, then `asExpr()` will turn it back into an AST expression, if possible.
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

## Acknowledgements

This is a reduced version of a Capture-the-Flag challenge devised by @esbena, available at https://securitylab.github.com/ctf/jquery. Try out the full version!
