<!-- -*- coding: utf-8 -*- -->
<!-- https://gist.github.com/hohn/
 -->
# CodeQL チュートリアル for C/C++: データフローとSQLインジェクション

<!--
 !-- xx:
 !-- md_toc github <  codeql-dataflow-sql-injection.md 
  -->

- [CodeQL Tutorial for C/C++: Data Flow and SQL Injection](#codeql-tutorial-for-cc-data-flow-and-sql-injection)
  - [セットアップ手順](#%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97%E6%89%8B%E9%A0%86)
  - [参考資料](#%E5%8F%82%E8%80%83%E8%B3%87%E6%96%99)
  - [Codeql Recap](#codeql-recap)
    - [from, where, select](#from-where-select)
    - [Predicates](#predicates)
    - [Existential quantifiers (local variables in queries)](#existential-quantifiers-local-variables-in-queries)
    - [Classes](#classes)
  - [The Problem in Action](#the-problem-in-action)
  - [Problem Statement](#problem-statement)
  - [Data flow overview and illustration](#data-flow-overview-and-illustration)
  - [Tutorial: Sources, Sinks and Flow Steps](#tutorial-sources-sinks-and-flow-steps)
    - [The Data Sink](#the-data-sink)
    - [The Data Source](#the-data-source)
    - [The Extra Flow Step](#the-extra-flow-step)
  - [The CodeQL Taint Flow Configuration](#the-codeql-taint-flow-configuration)
    - [Taint Flow Configuration](#taint-flow-configuration)
    - [Path Problem Setup](#path-problem-setup)
    - [Path Problem Query Format](#path-problem-query-format)
  - [Tutorial: Taint Flow Details](#tutorial-taint-flow-details)
    - [The isSink Predicate](#the-issink-predicate)
    - [The isSource Predicate](#the-issource-predicate)
    - [The isAdditionalTaintStep Predicate](#the-isadditionaltaintstep-predicate)
  - [Appendix](#appendix)
    - [The complete Query: SqlInjection.ql](#the-complete-query-sqlinjectionql)
    - [The Database Writer: add-user.c](#the-database-writer-add-userc)

## セットアップ手順

dotnet/coreclrに対してCodeQLのクエリを実行するためのステップです。

1. Visual Studio Code IDEをインストールします
2. [CodeQL extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql)をダウンロード＆インストールします。完全な手順は[here](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)になります。
3. ここで使用するCodeQL データベースは[`codeql-dataflow-sql-injection-d5b28fb.zip`](https://drive.google.com/file/d/1eBZ69ZQx6YnnZu41iUL0m8_e9qyMCZ9B/view?usp=sharing)からダウンロードできます。

4. Visual Studio Codeに先ほどダウンロードしたデータベースをインポートします:
    - Visual Studio Codeの左側のサイドバーにある **CodeQL** アイコンをクリックします
    - マウスを **Databases**のarchiveアイコンをクリックすると、File ExploreのWindowが開く
    - 当該データベースを選択します

5. CodeQLのクエリを作成するために、`cpp`ディレクトリの下で、本クエリようの　qlpackを作成します。
    - `codeql pack init sqlinjection -d .`を実行します
      (ノート：pack名はすべて小文字)
    -  `sqlinjection`ディレクトリにqlpack.ymlができるので、最後の行に以下を追加します。
    ```
       dependencies:
         codeql/cpp-all: "*"
    ```
6. Visual Studio Codeの`コマンドパレット`から`CodeQL: Install Pack Dependencies`を実行。選択画面から、`sqlinjection`にチェックして`OK`を押下。

7. `sqlinjection`ディレクトリの下に、`SqliInjection.ql`ファイルを作成。


## 参考資料 
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://help.semmle.com/QL/learn-ql)
- [Learning CodeQL for C/C++](https://help.semmle.com/QL/learn-ql/cpp/ql-for-cpp.html)
- [Using the CodeQL extension for VS Code](https://help.semmle.com/codeql/codeql-for-vscode.html)

## Codeql Recap
This is a brief review of CodeQL taken from the [full
introduction](https://git.io/JJqdS).  For more details, see the [documentation%E5%8F%82%E8%80%83%E8%B3%87%E6%96%99
links](#documentation-links).  We will revisit all of this during the tutorial.

### from, where, select(クエリの基本の３句)
CodeQLは叙述型言語です。クエリの結果を表示するために`select`を使って記述します。
一番単純な例:

```ql
import cpp

select "hello world"
```

もうちょっと複雑な例:
```ql
from /* ... variable declarations ... */
where /* ... logical formulas ... */
select /* ... expressions ... */
```

クエリの中で使用する変数は、`from`句で指定します。それら変数に対して制約をかけるときに`where`句で指定します。制約は複数指定が可能です。そして、その結果を`select`句を使って表示します。

`from`句の中に指定する変数は、複数指定できます。指定の方法は、type(型),variable(変数名)で指定します。
例：

```ql
from IfStmt ifStmt
select ifStmt
```

こちらの例では、typeとして、`IfStmt`、variableとして、`ifStmt`として`from`句で宣言しています。型の制約にマッチした値が変数に設定されます。ここでは、すべての`if`文を変数`ifStmt`に入っています。実行してみると、`if`文を抽出できることが確認できます。

次の例は、`if`ブロックの中から、空のブロックを抽出するクエリです：
```ql
from IfStmt ifStmt, BlockStmt block
where
  ifStmt.getThen() = block and
  block.getNumStmt() = 0
select ifStmt, "Empty if statement"
```


### Predicate
その他の特徴は、_predicate_です。ロジックの一部をカプセル化するために指定します。シンプルなクエリ`from`-`where`-`select`を考えます。このクエリは、タプル、もしくは、テーブルの行をアウトプットします。

空のブロックを検出する部分を新たにpredicateにする例を紹介します。predicateにすることで、別のクエリの中で再利用することができます。

```ql
predicate isEmptyBlock(Block block) {
  block.getNumStmt() = 0
}

from IfStmt ifStmt
where isEmptyBlock(ifStmt.getThen())
select ifStmt, "Empty if statement"
```

### Existential quantifiers (クエリ内のローカル変数)
*existential qualifiers*は関連のある状態を一時的変数に入れる簡単な方法です。文法例：

```ql
exists(<variable declarations> | <formula>)
```

existsの中の文法は、`from`と`where`句の構造に似ています。最初の部分は、変数を宣言して、二番目に変数を適用する式("conditions")を指定します。

例えば、クエリをリファクタリングするために活用します。
```ql
from IfStmt ifStmt, BlockStmt block
where
  ifStmt.getThen() = block and
  block.getNumStmt() = 0
select ifStmt, "Empty if statement"
```

空のブロックのために、一時的変数を使います。:
```ql
from IfStmt ifStmt
where
  exists(Block block |
    ifStmt.getThen() = block and
    block.getNumStmt() = 0
  )
select ifStmt, "Empty if statement"
```

1つのクエリからpredicateへはよく利用されます。

### Class
classはCodeQLの中で、新しい型を定義するときに利用します。再利用や、構造化コードでの利用が代表的です。

CodeQLでデフォルトで持っているあらゆる型のように、色々な値のひとつのまとまりを表現しています。実際、`BlockStmt`はclassであらゆるブロックを表現しています。それから、すべてのロジカルコンディションの定義を１つのclassとしてまとめることができます。

例として、空のブロックを表現する新たなclassを作成します。
```ql
class EmptyBlock extends Block {
  EmptyBlock() {
    this.getNumStmt() = 0
  }
}
```

そして、新しいclassの使用例を以下に示します。:
```ql
from IfStmt ifStmt, EmptyBlock block
where ifStmt.getThen() = block
select ifStmt, "Empty if statement"
```

## 問題の再現  
SQLインジェクションを実際のコードを実行することで体験します。


これを再現するためには、プログラムのビルドと、DBを作成するためにsqliteが必要になります。
以下の操作で、プログラムのビルド、DB構築します。
```sh
# Build
./build.sh

# Prepare db
./admin -r
./admin -c 
./admin -s
```

2つの方法で標準入出力からユーザをDBに登録します。2番目は、`echo`コマンドを使って、マニュアルの操作を代替します。

```sh
# Add regular user interactively
./add-user 2>> users.log
First User

# Regular user via "external" process
echo "User Outside" | ./add-user 2>> users.log
```

DBとlogを確認します:
```
# Check
./admin -s

tail -4 users.log 
```

結果は期待した通りになったと思います。:
```
0:$ ./admin -s
87797|First User
87808|User Outside

0:$ tail -4 users.log 
[Tue Jul 21 14:15:46 2020] query: INSERT INTO users VALUES (87797, 'First User')
[Tue Jul 21 14:17:07 2020] query: INSERT INTO users VALUES (87808, 'User Outside')
```

入力として、１つのテーブルそ想定して、削除するといった悪意ある入力がある可能性があります:
```sh
# Add Johnny Droptable 
./add-user 2>> users.log
Johnny'); DROP TABLE users; --
```

テーブルのコンテンツを確認してみます:
```sh
# And the problem:
./admin -s
0:$ ./admin -s
Error: near line 2: no such table: users
```

ログから、起こったことを考えてみます。:
```
1:$ tail -4 users.log 
[Tue Jul 21 14:15:46 2020] query: INSERT INTO users VALUES (87797, 'First User')
[Tue Jul 21 14:17:07 2020] query: INSERT INTO users VALUES (87808, 'User Outside')
[Tue Jul 21 14:18:25 2020] query: INSERT INTO users VALUES (87817, 'Johnny'); DROP TABLE users; --')
```

危険な入力データが,DBへ書き込むコマンドを実行できることが理解できたかと思います。つまり、入力データによりデータフローが悪いデータで汚染されたように見えます。クエリは、その汚染されたデータフローを見つけることです。:

## セキュリティ脆弱性問題解説 

この例は、SQLインジェクションです。ソースであるユーザ入力から、シンクはSQLクエリを処理するコードです。

ユーザが指定した文字列の一部が、攻撃者に、SQL分を挿入する隙をつくります。例えば、テーブル削除であったり、内部のデータを公開するなどです。

文字列連結を使ったSQLクエリを構成するソースコードを分析するためにCodeQLを使います。それからそのクエリを実行します。以下の例では、`sqlite3`ライブラリを使っています
- `stdin`で入力したデータを受信して、変数`buf`に格納します
- `snprintf`を使って、テーブルに`id`に対して、`buf`に格納されているデータを挿入
- `sqlite3_exec`からクエリを実行 

これは意図して簡単なコードにしていますが、実際のコードでも考慮しなければいけないです。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <ctype.h>
#include <sqlite3.h>
#include <time.h>

void write_log(const char* fmt, ...);

void abort_on_error(int rc, sqlite3 *db);

void abort_on_exec_error(int rc, sqlite3 *db, char* zErrMsg);
    
char* get_user_info() {
#define BUFSIZE 1024
    char* buf = (char*) malloc(BUFSIZE * sizeof(char));
    int count;
    // Disable buffering to avoid need for fflush
    // after printf().
    setbuf( stdout, NULL );
    printf("*** Welcome to sql injection ***\n");
    printf("Please enter name: ");
    count = read(STDIN_FILENO, buf, BUFSIZE);
    if (count <= 0) abort();
    /* strip trailing whitespace */
    while (count && isspace(buf[count-1])) {
        buf[count-1] = 0; --count;
    }
    return buf;
}

int get_new_id() {
    int id = getpid();
    return id;
}

void write_info(int id, char* info) {
    sqlite3 *db;
    int rc;
    int bufsize = 1024;
    char *zErrMsg = 0;
    char query[bufsize];
    
    /* open db */
    rc = sqlite3_open("users.sqlite", &db);
    abort_on_error(rc, db);

    /* Format query */
    snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
    write_log("query: %s\n", query);

    /* Write info */
    rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);
    abort_on_exec_error(rc, db, zErrMsg);

    sqlite3_close(db);
}

int main(int argc, char* argv[]) {
    char* info;
    int id;
    info = get_user_info();
    id = get_new_id();
    write_info(id, info);
    /*
     * show_info(id);
     */
}

```

source(ソース)とsink(シンク),それからセキュリティ脆弱性の起こるフローに注目して、具体的な問題を解決するためのロジックを示す：
In terms of sources, sinks, and information flow, the concrete problem for codeql is:
1. **source**として`buf`を指定している
2. **sink**として、`sqlite3_exec()｀へ渡す`cuery(クエリ)`を指定している
3. codeql ライブラリに関するいくつかの特定コードを指定している
3. sourceとsinkの間で、危険な、悪意あるフローパスを検出するCodeQLのフローライブラリを使用している


## データフローの概要とイラスト
前のセッションで、問題が起こりうる、関連する文字列である。`buf`をsource、`sqlite3_exec`をsinkと認識しました。

次に、sourceとsinkの間のデータフローを確認します。

そのソリューションは、データフローライブラリを使うことです。データフローは名前の通り、プログラムの中のデータフローを追跡することです。このようなセキュリティ脆弱性を見つける際、通常、その脆弱性のある場所から遡って、どこからそのデータが来たのか、自身で解析する際問うと思います。まさにそれをCodeQLのクエリとして実装しているのです。

方向性を持つグラフを使って、問題を探すデータフローを視覚化することができます。プログラムの中から、どこにノードがあるのか、フローのエッジ(境界)はどこにあるのかを視覚化します。そのパスが見つかれば、その２つの間のデータフローが明らかになります。

このグラフは、悪意あるデータに感染したパラメタからのデータフローを表します。グラフのノードは、関数のパラメタや、式のような値を持つプログラム要素です。このグラフのエッジ(境界)は、これらノードを通したフローを表しています。

CodeQLにおいて、２種類のデータフローがあります。
 - ローカル("intra-procedual")データフロー：１つの関数内で、データフローをモデル化。すべての関数に関して処理することができます。 
 - グローバル("inter-procedual")データフロー：すべての関数呼び出しを通してモデルか。すべての関数を解析できません。

グローバルデータフローがすべての関数に対して解析できない理由は、データフローパスの数が、指数関数的になるためです。

グローバルデータフローのこの問題を回避するために、_source_と_sink_が適用できるクエリを絞ることです。制限したパスのみを解析します。

このワークショップの説明の補足として[collection of slides](https://drive.google.com/file/d/1eEG0eGVDVEQh0C-0_4UIMcD23AWwnGtV/view?usp=sharing)の中でイラストで説明しています。

## Tutorial: Sources, Sinks and Flow Steps
<!--
XX:
 !-- The complete project can be downloaded via this 
 !-- [drive](https://drive.google.com/file/d/1-6c3S-e4FKa_IsuuzhhXupiAwCzzPgD-/view?usp=sharing)
 !-- link.
  -->

このセッションでは、２つのステップに分けて、知っていただきたいコンセプトを紹介します。今回のクエリは、実際に提供しているクエリをベースにしていますので、体験いただけることで、より一層理解が深まるかと思います。また、Visual Studio Codeの自動補完機能、および　jump-to-definitonコマンドで、実装部を参照できますので、理解を助けるものになります。

### データのシンク(sink)

`Function`の`getName`属性を使って、`sqlite3_exec`をまず検出します。

```ql
from Function f
where f.getName() = "sqlite3_exec" 
select f
```

1つ検出されるかと思います。
This should find one result, 
```ql
SQLITE_API int sqlite3_exec(
  sqlite3*,                                  /* An open database */
  const char *sql,                           /* SQL to be evaluated */
  int (*callback)(void*,int,char**,char**),  /* Callback function */
  void *,                                    /* 1st argument to callback */
  char **errmsg                              /* Error msg written here */
);
```
`FunctionCall`型を使って`sqlite3_exec`を呼び出している箇所を見つけます。
```ql
from FunctionCall exec
where exec.getTarget().getName() = "sqlite3_exec" 
select exec
```

実行すると、`add-user.c`の中の以下にインストラクションを検出します。

    rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);

その関数に渡される`query`パラメタが問題であるので、`getArgument`を使って取得します。
```ql
from FunctionCall exec, Expr query
where
    exec.getTarget().getName() = "sqlite3_exec" and
    query = exec.getArgument(1)
select exec, query
```

### データソース(source) 

外部からデータを読み取る場合、次のインストラクションを使います。

    count = read(STDIN_FILENO, buf, BUFSIZE);

`read`関数のパラメタ`buf`にデータが格納されるので、クエリでそれを取得するようにします。

```ql
from FunctionCall read, Expr buf
where
    read.getTarget().getName() = "read" and
    buf = read.getArgument(1)
select read, buf
```

### The Extra Flow Step
CodeQLデータフローライブラリは、ソースコードを極めて詳細に*可視化*します。しかし、不明瞭な関数を使っている場合においては、追加の実装が必要になります。
マクロのようにヘッダーだけに定義がある場合、関数として認識できないです。一例として、`snprintf`を呼び出すコードについて考えてみます。`snprintf`を呼び出すコードを実装したい場合、CodeQLは２つの`inflow`と`outflow`のノードを抽出します。
まずは、`Function`ライブラリを使って、`snprintf`を検出するクエリを作成します。
```ql
from FunctionCall printf
where printf.getTarget().getName() = "snprintf"
select printf
```

実行するとわかる通り、何も検出されません。一旦、ソースコード`add-user,c`と定義しているヘッダーを見てみます。
```c
#undef snprintf
#define snprintf(str, len, ...) \
  __builtin___snprintf_chk (str, len, 0, __darwin_obsz(str), __VA_ARGS__)
#endif
```

Cのマクロを使って、`__builtin___snprintf_chk`関数を`snprintf`で置き換えています。
`Function`では、Cマクロを検出できないため、その代わりに、`.matches`を使って名前のパターンマッチングを行います。:
```ql
from FunctionCall printf
where printf.getTarget().getName().matches("%snprintf%")
select printf
```

上記クエリを実行すると、以下のように期待した`snprintf`関数を検出できます。

    snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);

次のステップは、`inflow`と`outflow`についてです。`query`は`outflow`、`info`が`inflow`になります。    

`snprintf`マクロコールの中で、０から４までの引数があります。`__builtin__snprintf_chk`の関数の中では、０から６までのパラメタを指定できます。
In the `snprintf` macro call, those have indices 0 and 4.  In the underlying function
`__builtin___snprintf_chk`, the indices are 0 and 6.  Using the latter:
```ql
from FunctionCall printf, Expr out, Expr into
where
    printf.getTarget().getName().matches("%snprintf%") and
    printf.getArgument(0) = out and
    printf.getArgument(6) = into
select printf, out, into
```

上記が正しい呼び出しを検出し、拡張されたフローの引数になります。

<!-- !-- Practice exercise: !-- Very specific: shifted index for macro.
 Generalize this to consider !-- all trailing arguments as sources.  -->


エクササイズ: これまでみてきた例はmacosのライブラリでした。linuxでも、windowsでも動作するように、このクエリを汎用的に使えるような実装を考えてみましょう。ここ汎用化するために、`or`を使います。:

```ql
printf.getTarget().getName().matches("%snprintf%") and
(
  // mac version
or
 // linux version
or
 // windows version
)
```



##  危険なデータに汚染されたフローの設定
ここまでのクエリは、ソース、シンク、フローステップを抽出するものでした。グローバルデータフローと危険なデータを追跡するためには追加の処理が必要です。
 - 汚染データフローのフレームワークを作成
 - `path-problem`ヘッダと`semmle.code.cpp.dataflow.TaintTracking`のimportの追加
 - `path-problem`向けのクエリフォーム

これらの処理は次のセッションで実施します。

### 汚染データフローフレームワーク設定 
グローバル汚染データフローの設定は、`TaintTracking::Configuration`クラスの派生クラスを作成することです。そして、そのクラス内で、`isSource`,`isAdditionalTaintStep`predicateをオーバーライドすることです。

`source(ソース)`と`sink(シンク)`については、これまでのセッションの中ですでに説明した通りです。データフローと汚染データの追跡設定('TaintTracking::Configuration`)のクラスは、データフローパスを構築し、探す処理を実装する追加の機能です。

このステップが追加の汚染データフローを検出するステップとして追加しまステップとして追加します。デフォルトでこのような汚染データフローをモデル化していない場合に、追加することで、検出できるようになります。具体的には、`isAdditionalTaintStep`predicateを実装することです。このpredicateは`from`と`to`ノード(node)と２つのパラメタを持ち、汚染データフローもしくは、データフローグラグに拡張した2つの要素間のフローを追加します。

`configuration`の開始は、次のようになります。
in.

```ql
class SqliFlowConfig extends TaintTracking::Configuration {
    SqliFlowConfig() { this = "SqliFlow" }

    override predicate isSource(DataFlow::Node source) {
        // count = read(STDIN_FILENO, buf, BUFSIZE);
    }

    override predicate isSanitizer(DataFlow::Node sanitizer) { none() }

    override predicate isAdditionalTaintStep(DataFlow::Node into, DataFlow::Node out) {
        // Extra taint step for 
        //     snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
    }

    override predicate isSink(DataFlow::Node sink) {
        // rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);
    }
}
```

`TaintTracking::Configuration`は、_configuration_ クラスです。特定のpredicateに識別される１つのクラスインスタンスです。それから。プログラムの中の特定のソースを識別するために`isSource`predicate、特定のシンクを識別するために、`isSink`をオーバライドします。

### Path-Problem のセットアップ
デフォルトのクエリは、ソースとシンクのみをリストしただけです。結果を検査して、動作するために、ソースからシンクへのデータパスも必要です。これを解決するために、クエリは、_path problem_クエリのフォームに従って記述する必要があります。

具体的には、作成するクエリのヘッダの修正と、追加importが必要です。
 - `@kind`へのコメントは、`path-problem`に変更する。これにより、CodeQLのツールチェーンに、このクエリは、パスを結果として処理するよう知らせます。
 - 新たに`DataFlow::PathGraph`をimportすることで、クエリの結果として、パスデータを出力します。

`path-problem`と`TaintTacking::Configuration`ヘッダ、importを一緒に実装したものを以下に示します。
```ql
/**
 * @name SQLI Vulnerability
 * @description Using untrusted strings in a sql query allows sql injection attacks.
 * @kind path-problem
 * @id cpp/SQLIVulnerable
 * @problem.severity warning
 */

import cpp
import semmle.code.cpp.dataflow.TaintTracking
import DataFlow::PathGraph
```

### Path-Problem クエリ形式 
新しい`configuration`と`PathGraph`を利用するために、`hasFlowPath(source.sink)`predicateをコールします。これにより、定義したsource(ソース)とsink(シンク)との間で、該当するものを探す。ソースからシンクのグラフ探索アルゴリズムを実行していると考えてください。具体的クエリの実装は次のようになります。:

```ql
from SqliFlowConfig conf, DataFlow::PathNode source, DataFlow::PathNode sink
where conf.hasFlowPath(source, sink)
select sink, source, sink, "Possible SQL injection"
```

## チュートリアル: 汚染データフローの詳細 
データフローコンフィグレーションを使って、source(s),sink(s), taint step(s)の具体的な実装について、このセッションで説明します。

ここでのステップは、データフローの中で、前回のクエリを置き換える必要が出てきます。

### The isSink Predicate
これまでのクエリは、`Expr`ノードを使っていましたが、汚染データフローを検出するクエリは、`DataFlow::Node`を使います。

復習になりますが、次のクエリで、`sqlite3_exec`のコールのパラメタを抽出しました。

```ql
from FunctionCall exec, Expr query
where
    exec.getTarget().getName() = "sqlite3_exec" and
    query = exec.getArgument(1)
select exec, query
```

まず最初に、`DataFlow::Node`を組み込みます。`node`式を使うために、`node.asExpr()`を使うことがポイントです。

```ql
import cpp
import semmle.code.cpp.dataflow.TaintTracking

from FunctionCall exec, Expr query, DataFlow::Node sink
where
    exec.getTarget().getName() = "sqlite3_exec" and
    query = exec.getArgument(1) and
    sink.asExpr() = query
select exec, query, sink
```
`query`変数は冗長なので、簡素化します。
```ql
from FunctionCall exec, DataFlow::Node sink
where
    exec.getTarget().getName() = "sqlite3_exec" and
    sink.asExpr() = exec.getArgument(1) 
select exec, sink
```

次に、１つのパラメタを持つpredicate `isSink(DataFloe::Node sink)`を作成します。先ほどのクエリをpredicateにするためには、`exists()`[quantifier](https://help.semmle.com/QL/ql-handbook/formulas.html?highlight=exists#exists)を使って、クエリの中にある`FunctionCall exec`を`exists()`へ移動します。:

```ql
from DataFlow::Node sink
where
    exists(FunctionCall exec |
        exec.getTarget().getName() = "sqlite3_exec" and
        sink.asExpr() = exec.getArgument(1)
    )
select sink
```

それから、これをpredicateへ移動するために、`from`に指定している変数は、パラメタとなり、`where`句は、`exists()`のボディとなり、`select`は削除します。

```ql
predicate isSink(DataFlow::Node sink) {
    // rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);
    exists(FunctionCall exec |
        exec.getTarget().getName() = "sqlite3_exec" and
        sink.asExpr() = exec.getArgument(1)
    )
}
```

### The isSource Predicate
readコールへのパラメタ`buf`に外部から入力されたデータが格納されます。

    count = read(STDIN_FILENO, buf, BUFSIZE);

変数`query`を使って取得していました。

```ql
from FunctionCall read, Expr buf
where
    read.getTarget().getName() = "read" and
    buf = read.getArgument(1)
select read, buf
```
前回のセクションの中で、`isSource`predicateについては、１つのパラメタを渡す`predicate isSource(DataFlow::Node source)`に変換する必要があります。同じステップに従って、`DataFlow::Node`と`exists()`を使います。:

```ql
import cpp
import semmle.code.cpp.dataflow.TaintTracking

from DataFlow::Node source
where
    exists(FunctionCall read |
        read.getTarget().getName() = "read" and
        read.getArgument(1) = source.asExpr()
    )
select source
```

これに関しては、さらに調整が必要です。パラメタ`buf`は、`snprintf`関数コールにより、読み込まれ、書かれます。なぜなら、それを*source*として指定してますし、コール*後(after)*の値にもなるからです。post-updateノードへ[casting](https://help.semmle.com/QL/ql-handbook/expressions.html#casts)することで、この値を取得します。`source.asExpr()`の代わりに、`source.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr()`を使います。

最後に、predicateにこの実装を組み込みます。:

```ql
predicate isSource(DataFlow::Node source) {
    // count = read(STDIN_FILENO, buf, BUFSIZE);
    exists(FunctionCall read |
        read.getTarget().getName() = "read" and
        read.getArgument(1) = source.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr()
    )
}
```

quick-eval機能を使えば、`source`が今、`buf`ではなく、`ref arg buf`であることがわかります。

### The isAdditionalTaintStep Predicate
おさらいで、前回のクエリである、`snprintf`コールと、それに渡される０番目と六番目のパラメタを取得するクエリを再度示します。:

```ql
from FunctionCall printf, Expr out, Expr into
where
    printf.getTarget().getName().matches("%snprintf%") and
    printf.getArgument(0) = out and
    printf.getArgument(6) = into
select printf, out, into
```
`isSource`,`isSink`predicateについては、次のように変更する。
- `Expr`から`DataFlow::Node`へ変更
- outflow (`out`) 型を`PostUpdateNode`へ変更
- predicateへ変更

上記の変更内容を１つに実装します :

```ql
import cpp
import semmle.code.cpp.dataflow.TaintTracking

predicate isAdditionalTaintStep(DataFlow::Node into, DataFlow::Node out) {
    // Extra taint step for
    //     snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
    exists(FunctionCall printf |
        printf.getTarget().getName().matches("%snprintf%") and
        printf.getArgument(0) = out.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr() and
        printf.getArgument(6) = into.asExpr()
    )
}
```

## 補足説明
これまでの演習に使ったCのコードと、codeqlのクエリの完成形を載せます。

### The complete Query: SqlInjection.ql
クエリの完成形は次のようになります。

```ql
/**
 * @name SQLI Vulnerability
 * @description Using untrusted strings in a sql query allows sql injection attacks.
 * @kind path-problem
 * @id cpp/SQLIVulnerable
 * @problem.severity warning
 */

import cpp
import semmle.code.cpp.dataflow.TaintTracking
import DataFlow::PathGraph

class SqliFlowConfig extends TaintTracking::Configuration {
    SqliFlowConfig() { this = "SqliFlow" }

    override predicate isSource(DataFlow::Node source) {
        // count = read(STDIN_FILENO, buf, BUFSIZE);
        exists(FunctionCall read |
            read.getTarget().getName() = "read" and
            read.getArgument(1) = source.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr()
        )
    }

    override predicate isSanitizer(DataFlow::Node sanitizer) { none() }

    override predicate isAdditionalTaintStep(DataFlow::Node into, DataFlow::Node out) {
        // Extra taint step
        //     snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
        // But snprintf is a macro on mac os.  The actual function's name is
        //     #undef snprintf
        //     #define snprintf(str, len, ...) \
        //       __builtin___snprintf_chk (str, len, 0, __darwin_obsz(str), __VA_ARGS__)
        //     #endif
        exists(FunctionCall printf |
            printf.getTarget().getName().matches("%snprintf%") and
            printf.getArgument(0) = out.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr() and
            // very specific: shifted index for macro.
            printf.getArgument(6) = into.asExpr()
        )
    }

    override predicate isSink(DataFlow::Node sink) {
        // rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);
        exists(FunctionCall exec |
            exec.getTarget().getName() = "sqlite3_exec" and
            exec.getArgument(1) = sink.asExpr()
        )
    }
}

from SqliFlowConfig conf, DataFlow::PathNode source, DataFlow::PathNode sink
where conf.hasFlowPath(source, sink)
select sink, source, sink, "Possible SQL injection"
```

### The Database Writer: add-user.c
Cのコードになります。sqliteデータベースへの書き込みを行うコードです。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <ctype.h>
#include <sqlite3.h>
#include <time.h>

void write_log(const char* fmt, ...) {
    time_t t;
    char tstr[26];
    va_list args;

    va_start(args, fmt);
    t = time(NULL);
    ctime_r(&t, tstr);
    tstr[24] = 0; /* no \n */
    fprintf(stderr, "[%s] ", tstr);
    vfprintf(stderr, fmt, args);
    va_end(args);
    fflush(stderr);
}

void abort_on_error(int rc, sqlite3 *db) {
    if( rc ) {
        fprintf(stderr, "Can't open database: %s\n", sqlite3_errmsg(db));
        sqlite3_close(db);
        fflush(stderr);
        abort();
    }
}

void abort_on_exec_error(int rc, sqlite3 *db, char* zErrMsg) {
    if( rc!=SQLITE_OK ){
        fprintf(stderr, "SQL error: %s\n", zErrMsg);
        sqlite3_free(zErrMsg);
        sqlite3_close(db);
        fflush(stderr);
        abort();
    }
}
    
char* get_user_info() {
#define BUFSIZE 1024
    char* buf = (char*) malloc(BUFSIZE * sizeof(char));
    int count;
    // Disable buffering to avoid need for fflush
    // after printf().
    setbuf( stdout, NULL );
    printf("*** Welcome to sql injection ***\n");
    printf("Please enter name: ");
    count = read(STDIN_FILENO, buf, BUFSIZE);
    if (count <= 0) abort();
    /* strip trailing whitespace */
    while (count && isspace(buf[count-1])) {
        buf[count-1] = 0; --count;
    }
    return buf;
}

int get_new_id() {
    int id = getpid();
    return id;
}

void write_info(int id, char* info) {
    sqlite3 *db;
    int rc;
    int bufsize = 1024;
    char *zErrMsg = 0;
    char query[bufsize];
    
    /* open db */
    rc = sqlite3_open("users.sqlite", &db);
    abort_on_error(rc, db);

    /* Format query */
    snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
    write_log("query: %s\n", query);

    /* Write info */
    rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);
    abort_on_exec_error(rc, db, zErrMsg);

    sqlite3_close(db);
}

int main(int argc, char* argv[]) {
    char* info;
    int id;
    info = get_user_info();
    id = get_new_id();
    write_info(id, info);
    /*
     * show_info(id);
     */
}
    
```
