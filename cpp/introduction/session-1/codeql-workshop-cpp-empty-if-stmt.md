# CodeQL workshop for C/C++: Empty if statements

- 分析対象言語 : C/C++
- 難しさレベル level: 1/3

## セキュリティ脆弱性問題説明 

このワークショップでは、画像メタデータ管理用OSS C++ライブラリの１つ、[Exiv2](https://www.exiv2.org/)のソースコードを解析するためのクエリを作成します。

冗長なコードは、プログラマーがロジックを間違う原因になります。ここに簡単なコード例を示します。:

```c
int write(int buf[], int size, int loc, int val) {
    if (loc >= size) {
       // return -1;
    }

    buf[loc] = val;

    return 0;
}
```
バッファに値を挿入するCの関数です。オーバーフローを防止するチェックコードを含んでいます。しかし、状態を返すコードはコメントアウトされています。この場合、このコードは無意味で、境界のチェックがありません。

このワークショップでは、このような意味のない、冗長なコードを検出するクエリを見つけることが可能です。このクエリを進めるにあたっては、_seed vulnerability_が良い例になります。それは、他のインスタンス("variants")を見つけるクエリです。

このワークショップを実施すると次のことがわかります。:
 - クエリの基本構造
 - QL の型と変数
 - 論理状態

## Setup instructions

### Writing queries on your local machine

To run CodeQL queries on Exiv2, follow these steps:

1. Visual Studio Code IDEのインストール
2. [CodeQL extension for Visual Studio Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)のダウンロード＆インストール。 完全なセットアップ手順は、[here](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/)を参照下さい。
3. [Set up the starter workspace](https://codeql.github.com/docs/codeql-for-visual-studio-code/setting-up-codeql-in-visual-studio-code/#starter-workspace)をセットアップします。
    - **Important**: 標準のクエリライブラリもクローンするために、ローカルにクローンする際は、`git clone --recursive` もしくは `git submodule update --init --remote`を指定することを忘れないように。 
4. VSCodeを実行して、次のようにワークスペースをオープンします。: File > Open Workspace > `vscode-codeql-starter/vscode-codeql-starter.code-workspace`をブラウズします。
5. [Exiv2 database](http://downloads.lgtm.com/snapshots/cpp/exiv2/Exiv2_exiv2_b090f4d.zip)をダウンロードします。
6. Import the unzipped database into Visual Studio Code:
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the + sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.
7. Create a new file, name it `EmptyIf.ql`, save it under `codeql-custom-queries-cpp`.

## Documentation links
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://help.semmle.com/QL/learn-ql)
- [Learning CodeQL for C/C++](https://help.semmle.com/QL/learn-ql/cpp/ql-for-cpp.html)
- [Using the CodeQL extension for VS Code](https://help.semmle.com/codeql/codeql-for-vscode.html)

## Workshop
The workshop is split into several steps. You can write one query per step, or work with a single query that you refine at each step. Each step has a **hint** that describes useful classes and predicates in the CodeQL standard libraries for C/C++. You can explore these in your IDE using the autocomplete suggestions and jump-to-definition command.

The basic syntax of QL will look familiar to anyone who has used SQL. A query is defined by a _select_ clause, which specifies what the result of the query should be. For example:
```ql
import cpp

select "hello world"
```
This query simply returns the string "hello world".

More complicated queries look like this:
```ql
from /* ... variable declarations ... */
where /* ... logical formulas ... */
select /* ... expressions ... */
```
The `from` clause specifies some variables that will be used in the query. The `where` clause specifies some conditions on those variables in the form of logical formulas. The `select` clauses speciifes what the results should be, and can refer to variables defined in the `from` clause.

The `from` clause is defined as a series of variable declarations, where each declaration has a _type_ and a _name_. For example:
```ql
from IfStmt ifStmt
select ifStmt
```
We are declaring a variable with the name `ifStmt` and the type `IfStmt` (from the CodeQL standard library for analyzing C/C++). Variables represent a set of values, initially constrained by the type of the variable. Here, the variable `ifStmt` represents the set of all `if` statements in the C/C++ program, as we can see if we run the query.

To find empty if statements we will first need to define what it means for the if statement to be empty. If we consider the code sample again:
```c
    if (loc >= size) {
       // return -1;
    }
```
What we can see is that there is a _block_ (i.e. an area of code enclosed by `{` and `}`) which includes no executable code.

1. Write a new query to identify all blocks in the program.
    <details>
    <summary>Hint</summary>

    A block statement is represented by the standard library type `BlockStmt`.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from BlockStmt block
    select block
    ```
    </details>

QL is an _object-oriented_ language. One consequence of this is that "objects" can have "operations" associated with them. QL uses a "." notation to allow access to operations on a variable. For example:
```ql
from IfStmt ifStmt
select ifStmt, ifStmt.getThen()
```
This reports the "then" part of each if statement (i.e. the part that is executed if the conditional succeeds). If we run the query, we find that it now reports two columns, where the first column is if statements in the program, and the second column is the blocks associated with the "then" part of those if statements.

2. Update your query to report the number of statements within each block.
    <details>
    <summary>Hint</summary>

    `BlockStmt` has an operation called `getNumStmt()`.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from BlockStmt block
    select block, block.getNumStmt()
    ```
    </details>

We can apply further constraints to the variables by adding a `where` clause, and which specifies logical formula that describes additional conditions on the variables. 
For example:

```ql
from IfStmt ifStmt, BlockStmt block
where ifStmt.getThen() = block
select ifStmt, block
```
Here we have added another variable called `block`, and then added a condition (a "formula" in QL terms) to the `where` clause that states that the `block` is equal to the result of the `ifStmt.getThen()` operation. Note: the equals sign (`=`) here is _not_ an assignment - it is _declaring_ a condition on the variables that must be satisified for a result to be reported. Additional conditions can be provided by combining them with an `and`.

When we run this query, we again get two columns, with if statements and then parts, however we now get fewer results. This is because not all if statements have blocks as then parts. Consider:
```c
if (x)
  return 0;
```
This reveals a feature of QL - proscriptive typing. By specifying that the variable `block` has type "BlockStmt", we are actually asserting some logical conditions on that variable i.e. that it represents a block. In doing so, we have also limited the set of if statements to only those where the then part is associated with a block.

3. Update your query to report only blocks with `0` stmts.
    <details>
    <summary>Hint</summary>

    Add a `where` clause which states that the number of statements in the block is equal to `0`.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from BlockStmt block
    where block.getNumStmt() = 0
    select block
    ```
    </details>

4. Combine your query identifying "empty" blocks with the query identifying blocks associated with if statements, to report all empty if statements in the program.
    <details>
    <summary>Hint</summary>

    Add a new condition to the `where` clause using the logical connective `and`.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from IfStmt ifStmt, BlockStmt block
    where
      ifStmt.getThen() = block and
      block.getNumStmt() = 0
    select ifStmt, block
    ```
    </details>

Congratulations, you have now written your first CodeQL query that finds genuine bugs!

In order to use this with the rest of the CodeQL toolchain, we will need to make some final adjustments to the query to add metadata, to help the toolchain interpret and process the results.

<details>
<summary>Reveal</summary>

```ql
/**
 * @name Empty if statement
 * @kind problem
 * @id cpp/empty-if-statement
 */
import cpp

from IfStmt ifStmt, BlockStmt block
where
  ifStmt.getThen() = block and
  block.getNumStmt() = 0
select ifStmt, "Empty if statement"
```
</details>


5. _Optional homework_ Review the results. Are there any patterns which are causing false positives to appear? Can you adjust the query to reduce false positives?
