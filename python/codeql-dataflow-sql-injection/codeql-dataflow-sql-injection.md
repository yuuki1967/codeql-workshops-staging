<!-- -*- coding: utf-8 -*- -->
<!-- https://gist.github.com/hohn/
 -->
# CodeQL Tutorial for Python: データフローとSQL インジェクション  

<!--
 !-- xx:
 !-- md_toc github <  codeql-dataflow-sql-injection.md 
  -->

- [CodeQL Tutorial for C/C++: Data Flow and SQL Injection](#codeql-tutorial-for-cc-data-flow-and-sql-injection)
  - [Setup Instructions](#setup-instructions)
  - [Documentation Links](#documentation-links)
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

## Setup Instructions

To set up VS Code for CodeQL query development, follow these steps:

1. Visual Studio Code IDEのインストール
2. [CodeQL extension for Visual Studio Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)のダウンロード＆インストール。 完全なセットアップ手順は、[here](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)を参照下さい。
3. [Set up the starter workspace](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/#starter-workspace)をセットアップします。
    - **Important**: 標準のクエリライブラリもクローンするために、ローカルにクローンする際は、`git clone --recursive` もしくは `git submodule update --init --remote`を指定することを忘れないように。 
4. VSCodeを実行して、次のようにワークスペースをオープンします。: File > Open Workspace > `vscode-codeql-starter/vscode-codeql-starter.code-workspace`をブラウズします。
5. 今回対象のデータベース[`python-sqli-b0e435c.zip`](https://drive.google.com/file/d/1eoY-8nwdiIEZHLYIGnhIkbaVHQaQuGB0/view?usp=sharing)をダウンロード
6. Visual Studio Codeにデータベースを入れます:
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the + sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.
7. 新しくファイル`SqliInjection.ql`を作成し、`codeql-dataflow-sql-injection`ディレクトリの下に保存します。


## 参照資料 
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://codeql.github.com/docs/)
- [Learning CodeQL for Python](https://codeql.github.com/docs/codeql-language-guides/codeql-for-python/)
- [Using the CodeQL extension for VS Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)

## Codeql 概要
CodeQLのPyhton向けクエリの学習セッションです。このセッションを通じて、より詳しくクエリを知ることができます。さらなる詳細は、 [documentation links](#documentation-links)を参照ください。

### from, where, select
CodeQLは叙述型の言語で、基本的なクエリは、クエリの結果を指定する`select`句を定義します。例えば、:

```ql
import python

select "hello world"
```

より、複雑なクエリは、次のような例です：
```ql
from /* ... variable declarations ... */
where /* ... logical formulas ... */
select /* ... expressions ... */
```
クエリの中で使用する変数を指定は、`from`句で指定します。ロジック式の中でこれら変数に対して条件を指定する際に`where`句を使用します。`select`句が、結果を指定します。その際に、`from`句で定義した変数を参照することができます。

`from`句は、変数の宣言を繋げていきます。それぞれの宣言は、_type_ と_name_の順で指定します。
例を次に示します。：
```ql
from If ifStmt
select ifStmt
```
この例では、型(type)として`If`で、`ifStmt`という名前の変数を宣言しています。変数は、型と値で表現します。ここに、変数`ifStmt`がPythonプログラムの中のすべての`if`文が入っています。これを実行すれば、参照することができます。

空のブランチを見つけるクエリは次のようになります。上述した３つの句をすべて利用しています。:
```ql
import python

from If ifstmt
where
    count(ifstmt.getBody().getAnItem()) = 1 and
    ifstmt.getBody().getAnItem() instanceof Pass
select ifstmt
```


### Predicates
その他の特徴として、_predicates_があります。再利用するためにプログラムの中にロジックの一部を隠蔽する(抽象化する)ための方法を提供します。一番シンプルな`from`-`where`-`select`で、`select`に結果テーブルの中の行もしくは、タプル（複数の要素構成）も出力することができます。

空のブロックを抽出するクエリの中に新しいpredicateを紹介します。

```ql
import python

predicate isEmptyBlock(StmtList stmt) {
    count(stmt.getAnItem()) = 1 and
    stmt.getAnItem() instanceof Pass
}

from If ifstmt
where
    isEmptyBlock(ifstmt.getBody())
select ifstmt
```

### Existential quantifiers (クエリの中のローカル変数)
この言葉はロジック、ロジックプログラム独特なものです。*existential qualifiers*をわかりやすく説明すると、関連した状態を持った一時的な変数のことです。文法は：

```ql
exists(<variable declarations> | <formula>)
```

existsは`from`と`where`句と似た構造を持ちます。最初は、１つ以上のせんげん、２つ目は、それらの変数を適用した式("conditions")を指定します。

これを使ってこのクエリをリファクタすることができます。
```ql
import python

from If ifstmt
where
    count(ifstmt.getBody().getAnItem()) = 1 and
    ifstmt.getBody().getAnItem() instanceof Pass
select ifstmt
```

to use a temporary variable for the empty block:
```ql
import python

from If ifstmt
where
    exists(Stmt stmt |
        stmt = ifstmt.getBody().getAnItem() and
        count(stmt) = 1 and
        stmt instanceof Pass
    )
select ifstmt
```

このような例は、１つのクエリからpredicateへ変更する際のよく使われるテクニックです。ローカル変数を使うことで、predicateのインターフェースをよりコンパクトにすることができます。

### Classes
classsはCodeQLの中で新しい型を定義できます。こちらも、再利用、構造化コードをする際の簡単な手法を提供するものです。

Like all types in CodeQL, classes represent a set of values. For example, the
`StmtList` type is, in fact, a class, and it represents the set of all statement
lists in the program. You can also think of a class as defining a set of logical
conditions that specifies the set of values for that class.

For example, we can define a new CodeQL class to represent empty statement lists:
```ql
import python

class EmptyStmt extends StmtList {
    EmptyStmt() {
        count(this.getAnItem()) = 1 and
        this.getAnItem() instanceof Pass
    }
}
```

and use it in a query:
```ql
from If ifstmt, EmptyStmt empty
where
    ifstmt.getBody() = empty
select ifstmt
```

## 問題の再現
Running the code is a great way to see the problem and check whether the code is
vulnerable.

This program can be compiled and linked, and a simple sqlite db created via 

```sh
# Build
./build.sh

# Prepare db
./admin -r
./admin -c 
./admin -s
```

Users can be added via `stdin` in several ways; the second is a pretend "server"
using the `echo` command.

```sh
# Add regular user interactively
./add-user 2>> users.log
First User

# Regular user via "external" process
echo "User Outside" | ./add-user 2>> users.log
```

Check the db and log:
```
# Check
./admin -s

tail -4 users.log 
```

Looks ok:
```
0:$ ./admin -s
87797|First User
87808|User Outside

0:$ tail -4 users.log 
[Tue Jul 21 14:15:46 2020] query: INSERT INTO users VALUES (87797, 'First User')
[Tue Jul 21 14:17:07 2020] query: INSERT INTO users VALUES (87808, 'User Outside')
```

But there may be bad input; this one guesses the table name and drops it:
```sh
# Add Johnny Droptable 
./add-user 2>> users.log
Johnny'); DROP TABLE users; --
```

And then we have this:
```sh
# And the problem:
./admin -s
0:$ ./admin -s
Error: near line 2: no such table: users
```

What happened?  The log shows that data was treated as command:
```
1:$ tail -4 users.log 
[Tue Jul 21 14:15:46 2020] query: INSERT INTO users VALUES (87797, 'First User')
[Tue Jul 21 14:17:07 2020] query: INSERT INTO users VALUES (87808, 'User Outside')
[Tue Jul 21 14:18:25 2020] query: INSERT INTO users VALUES (87817, 'Johnny'); DROP TABLE users; --')
```

Looking ahead, we now *know* that there is unsafe external data (source)
which reaches (flow path) a database-writing command (sink).  Thus, a query
written against this code should find at least one taint flow path.

## Problem Statement

Many security problems can be phrased in terms of _information flow_:

_Given a (problem-specific) set of sources and sinks, is there a path in the data
flow graph from some source to some sink?_

The example we look at is SQL injection: sources are user-input, sinks are SQL
queries processing a string formed at runtime.

When parts of the string can be specified by the user, they allow an attacker to
insert arbitrary sql statements; these could erase a table or extract internal
data etc.

We will use CodeQL to analyze the source code constructing a SQL
query using string concatenation and then executing that query
string.  The following example uses the `sqlite3` library; it 
- receives user-provided data from `stdin` and keeps it in `buf`
- uses environment data and stores it in `id`,
- runs a query in `sqlite3_exec`

This is intentionally simple code, but it has all the elements that have to be
considered in real code and illustrates the QL features. 

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

In terms of sources, sinks, and information flow, the concrete problem for codeql is:
1. specifying `buf` as **source**,
2. specifying the `query` argument to `sqlite3_exec()` as **sink**, 
3. specifying some code-specific data flow steps for the codeql library,
3. using the codeql taint flow library find taint flow paths (if there are any)
   between the source and the sink.

In the following, we go into more concrete detail and develop codedql scripts to
solve this problem.



## Data flow overview and illustration
In the previous sections we identified the sources of problematic strings
(accesses of `info` etc.), and the sink that their data may flow to (the argument
to `sqlite3_exec`).

We need to see if there is data flow between the source(s) and this sink.

The solution here is to use the data flow library.  Data flow is, as the name
suggests, about tracking the flow of data through the program. It helps answers
questions like: does this expression ever hold a value that originates from a
particular other place in the program?

We can visualize the data flow problem as one of finding paths through a directed
graph, where the nodes of the graph are elements in program, and the edges
represent the flow of data between those elements. If a path exists, then the data
flows between those two nodes.

This graph represents the flow of data from the tainted parameter. The nodes of
graph represent program elements that have a value, such as function parameters
and expressions. The edges of this graph represent flow through these nodes.

There are two variants of data flow available in CodeQL:
 - Local (“intra-procedural”) data flow models flow within one function; feasible
   to compute for all functions in a CodeQL database.
 - Global (“inter-procedural”) data flow models flow across function calls; not
   feasible to compute for all functions in a CodeQL database.

While local data flow is feasible to compute for all functions in a CodeQL
database, global data flow is not. This is because the number of paths becomes
_exponentially_ larger for global data flow.

The global data flow (and taint tracking) library avoids this problem by requiring
that the query author specifies which _sources_ and _sinks_ are applicable. This
allows the implementation to compute paths only between the restricted set of
nodes, rather than for the full graph.

To illustrate the dataflow for this problem, we have a [collection of slides](https://drive.google.com/file/d/1eEG0eGVDVEQh0C-0_4UIMcD23AWwnGtV/view?usp=sharing)
for this workshop.

## Tutorial: Sources, Sinks and Flow Steps
<!--
XX:
 !-- The complete project can be downloaded via this 
 !-- [drive](https://drive.google.com/file/d/1-6c3S-e4FKa_IsuuzhhXupiAwCzzPgD-/view?usp=sharing)
 !-- link.
  -->

The tutorial is split into several steps and introduces concepts as they are
needed.  Experimentation with the presented queries is encouraged, and the
autocomplete suggestions (Ctrl + Space) and the jump-to-definition command (F12 in
VS Code) are good ways explore the libraries.


### The Data Sink
Now let's find the function `sqlite3_exec`.  In CodeQL, this uses `Function`
and a `getName()` attribute.

```ql
from Function f
where f.getName() = "sqlite3_exec" 
select f
```

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
in the header `sqlite3.h`.

Next, let's find the calls to `sqlite3_exec` using the `FunctionCall` type
```ql
from FunctionCall exec
where exec.getTarget().getName() = "sqlite3_exec" 
select exec
```

This finds our call in `add-user.c`, 

    rc = sqlite3_exec(db, query, NULL, 0, &zErrMsg);

We are interested in the `query` argument, which we can get using `.getArgument`:
```ql
from FunctionCall exec, Expr query
where
    exec.getTarget().getName() = "sqlite3_exec" and
    query = exec.getArgument(1)
select exec, query
```

### The Data Source

The external data enters through the call

    count = read(STDIN_FILENO, buf, BUFSIZE);

We thus want the `buf` argument to the call of the `read` function.  Together, this is 

```ql
from FunctionCall read, Expr buf
where
    read.getTarget().getName() = "read" and
    buf = read.getArgument(1)
select read, buf
```

### The Extra Flow Step
The codeql data flow library traverses *visible* source code fairly well, but flow
through opaque functions requires additional support (more on this later).
Functions for which only a headers is available are opaque, and we have one of
these here: the call to `snprintf`.  Once we locate this call, there are *two* nodes
to identify: the inflow and outflow.

Let's start with `snprintf`.  If we try
```ql
from FunctionCall printf
where printf.getTarget().getName() = "snprintf"
select printf
```
we get zero results.  This is puzzling; if we visit the `add-user.c` source and
follow the definition of `snprintf`, it turns out to be a macro on MacOS:
```c
#undef snprintf
#define snprintf(str, len, ...) \
  __builtin___snprintf_chk (str, len, 0, __darwin_obsz(str), __VA_ARGS__)
#endif
```

Fortunately, the underlying function `__builtin___snprintf_chk` has `snprintf` in
the name.  So instead of working with C macros from codeql, we generalize our
query using a name pattern with `.matches`:
```ql
from FunctionCall printf
where printf.getTarget().getName().matches("%snprintf%")
select printf
```

This identifies our call

    snprintf(query, bufsize, "INSERT INTO users VALUES (%d, '%s')", id, info);
    
and we need the inflow and outflow nodes next.  `query` is the outflow, `info` is
the inflow.

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

This correctly identifies the call and the extra flow arguments.

<!-- !-- Practice exercise: !-- Very specific: shifted index for macro.
 Generalize this to consider !-- all trailing arguments as sources.  -->


Practice exercise: If you are using linux or windows, generalize this query for
the `snprintf` arguments found there.  One way to do this is using `or`:

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



## The CodeQL Taint Flow Configuration
The previous queries identify our source, sink and one additional flow step.  To
use global data flow and taint tracking we need some additional codeql setup:
 - a taint flow configuration 
 - the path problem header and imports
 - a query formatted for path problems.

These are done next.

### Taint Flow Configuration
The way we configure global taint flow is by creating a custom extension of the
`TaintTracking::Configuration` class, and speciyfing `isSource`, `isSink`, and 
`isAdditionalTaintStep` predicates.

The sources and sinks were explained earlier.  Data flow and taint tracking
configuration classes support a number of additional features that help configure
the process of building and exploring the data flow path.

One such feature is adding additional taint steps. This is useful if you use
libraries which are not modelled by the default taint tracking. You can implement
this by overriding `isAdditionalTaintStep` predicate. This has two parameters, the
`from` and the `to` node, and it essentially allows you to add extra edges into the
taint tracking or data flow graph.

A starting configuration can look like the following, with details to be filled
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

`TaintTracking::Configuration` is a _configuration_ class. In this case, there will be
a single instance of the class, identified by a unique string specified in the
characteristic predicate. We then override the `isSource` predicates to represent
the set of possible sources in the program, and `isSink` to represent the possible
set of sinks in the program.

### Path Problem Setup
Queries will only list sources and sinks by default.  To inspect these results and
work with them, we also need the data paths from source to sink.  For this, the
query needs to have the form of a _path problem_ query.

This requires a modifications to the query header and an extra import: 
 - The `@kind` comment has to be `path-problem`. This tells the CodeQL toolchain
   to interpret the results of this query as path results. 
 - A new import `DataFlow::PathGraph`, which will report the path data
   alongside the query results. 

Together, this looks like
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

### Path Problem Query Format
To use this new configuration and `PathGraph` support, we call the
`hasFlowPath(source, sink)` predicate, which will compute a reachability table
between the defined sources and sinks.  Behind the scenes, you can think of this as
performing a graph search algorithm from sources to sinks.  The query will look
like this:

```ql
from SqliFlowConfig conf, DataFlow::PathNode source, DataFlow::PathNode sink
where conf.hasFlowPath(source, sink)
select sink, source, sink, "Possible SQL injection"
```

## Tutorial: Taint Flow Details
With the dataflow configuration in place, we just need to provide the details for
source(s), sink(s), and taint step(s).

Some more steps are required to convert our previous queries for use in data
flow.  These are covered here.

### The isSink Predicate
Note that our previous queries used `Expr` nodes, but the taint query requires
`DataFlow::Node` nodes.

We have identified arguments to the call of the `sqlite3_exec` function via the
query

```ql
from FunctionCall exec, Expr query
where
    exec.getTarget().getName() = "sqlite3_exec" and
    query = exec.getArgument(1)
select exec, query
```

First, we need to incorporate the `DataFlow::Node`.  The key to this is
`node.asExpr()`, which yields the `node`'s expression.  Adding this we get

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

Notice that `query` is now redundant, so this simplifies to 
```ql
from FunctionCall exec, DataFlow::Node sink
where
    exec.getTarget().getName() = "sqlite3_exec" and
    sink.asExpr() = exec.getArgument(1) 
select exec, sink
```

Second, we need this as a predicate of a single argument, `predicate
isSink(DataFlow::Node sink)`.  For this we introduce the `exists()`
[quantifier](https://help.semmle.com/QL/ql-handbook/formulas.html?highlight=exists#exists)
to move the `FunctionCall exec` into the body of the query and remove it from the
result:

```ql
from DataFlow::Node sink
where
    exists(FunctionCall exec |
        exec.getTarget().getName() = "sqlite3_exec" and
        sink.asExpr() = exec.getArgument(1)
    )
select sink
```

To turn this into a predicate, `from` contents become arguments, the `where`
becomes the body, and the `select` is dropped:

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
Recall that the external data enters through the `buf` argument to the call

    count = read(STDIN_FILENO, buf, BUFSIZE);

and we got this via the query

```ql
from FunctionCall read, Expr buf
where
    read.getTarget().getName() = "read" and
    buf = read.getArgument(1)
select read, buf
```

As for the `isSink` predicate in the previous section, we need to convert this to
a predicate of a single argument, `predicate isSource(DataFlow::Node source)`.
Following the same steps, we introduce a `DataFlow::Node` and an `exists()`:

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

There is one more adjustment needed for this to work.  The `buf` argument is both
read by and written to by the `snprintf` function call.  Because we are specifying
it as a *source*, the value of interest is the value *after* the call.  We get
this value by
[casting](https://help.semmle.com/QL/ql-handbook/expressions.html#casts) to the
post-update node.  Instead of `source.asExpr()`, we use
`source.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr()`


Last, we incorporate this into a predicate:

```ql
predicate isSource(DataFlow::Node source) {
    // count = read(STDIN_FILENO, buf, BUFSIZE);
    exists(FunctionCall read |
        read.getTarget().getName() = "read" and
        read.getArgument(1) = source.(DataFlow::PostUpdateNode).getPreUpdateNode().asExpr()
    )
}
```

If you quick-eval this predicate, you will see that `source` is now `ref arg buf`
instead of `buf`.


### The isAdditionalTaintStep Predicate
Our previous query identifies the call to `snprintf` and the extra flow arguments:

```ql
from FunctionCall printf, Expr out, Expr into
where
    printf.getTarget().getName().matches("%snprintf%") and
    printf.getArgument(0) = out and
    printf.getArgument(6) = into
select printf, out, into
```

As for the `isSource` and `isSink` predicates, we need to
- change from `Expr` to a `DataFlow::Node`
- change the outflow (`out`) type to a `PostUpdateNode`
- convert this to a predicate

Put together:

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

## Appendix
This appendix has the complete C source and codeql query.

### The complete Query: SqlInjection.ql
The full query is

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
The complete source for the sqlite database writer
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
