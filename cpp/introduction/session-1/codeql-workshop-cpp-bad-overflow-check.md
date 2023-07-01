# CodeQL workshop for C/C++: Bad overflow checks

- 分析対象言語 : C/C++
- 難しさレベル : 1/3

## セキュリティ脆弱性説明 

このワークショップでは、[ChakraCore](https://github.com/microsoft/ChakraCore)のソースコードをCodeQLで分析します。対象リポジトリは、Microsoft社が開発したOSSのJavaScriptエンジンです。

C/C++の算術オペレータ _overflow_の中で、そのオペレーションの結果定義した変数の型の範囲を超える大きな値を格納した場合を考えます。オペレーションがオーバフローした場合、大きい値から小さい値へラップします。デベロッパーは、オーバフロー検証するためのロジックを入れるために、この属性を利用します。

```c
if (v + b < v)
    handle_error("overflow");
} else {
    result = v + b;
}
```

`v + b`の結果がオーバフローしているかを確認するロジックです。`v`の値は、ラップはしていないですが、この計算結果はラップします。

この問題の間違いを探します。

<details>
<summary>Answer</summary>
CPUは通常、CPUのアーキテクチャは、最適化して、32bit以上の整数の算術演算を行います。コンパイラは、32bitよりも小さい算術演算に対して、"整数の拡張"を実行します。

これは、32bitよりも小さい型上で計算される場合は、期待したとおり、オーバーフローは起きません。大きい型を使って計算した場合にオーバーフローが起きます。

The arguments of the following arithmetic operators undergo implicit conversions:

- binary arithmetic * / % + -
- relational operators < > <= >= == !=
- binary bitwise operators & ^ |
- the conditional operator ?:

See <a href="https://en.cppreference.com/w/c/language/conversion">https://en.cppreference.com/w/c/language/conversion</a> for more details.
</details>

次のコードを例にします。:

```c
uint16_t v = 65535;
uint16_t b = 1;
uint16_t result;
if (v + b < v) {
    handle_error("overflow");
} else {
    result = v + b;
}
```
結果を見ます。
<details>
<summary>Answer</summary>
再度、コード例を示します。暗黙的に型変換が行われます。:

```c
uint16_t v = 65535;
uint16_t b = 1;
uint16_t result;
if ((int)v + (int)b < (int)v) {
    handle_error("overflow");
} else {
    result = (uint16_t)((int)v + (int)b);
}
```

この例では、16bitのオーバーフローが起きたとしても、二番目の分岐を実行します。結果は０になります。

説明:

-  2つの整数値`v`と`b`は、32bit整数拡張されます
- 比較演算子(`<`)もまた、算術演算です。そのため、32bitで演算します。 
-  `v + b < v`は、真にはなりません。`v`、`b`ともに最大2<sup>16</sup>であるため、32bitを超えることはないからです。
- そのため、常に二番目の分岐が実行され、足し算の結果が変数に代入されます。結果は、16bitで格納されるため、オーバフローはまだ解決できません。

端的に言うと、32bitよりも小さい型同士の足し算結果のオーバーフロー検証は無意味となります。
</details>

We will begin by using CodeQL to identify relational comparison operations (`<`, `>`, `<=`, `>=`) that look like overflow checks. We will then refine this to identify only those checks which are likely wrong due to integer promotion.

If you follow the workshop all the way to the end, then you will find a real security in an old version of ChakraCore.

## Setup instructions

### Writing queries on your local machine

To run CodeQL queries on ChakraCore offline, follow these steps:

1. Download the [ChakraCore database](https://drive.google.com/file/d/1Jhxylk0b6My3P61-nt3_DkHreAXQltWz/view?usp=sharing).
1. Import the database into Visual Studio Code:
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the archive sign that appears on the right.
    - Choose the database directory on your filesystem.
1. Create a new directory `bad-overflow-queries` in the Visual Studio Code Explorer view.
1. Add the file `qlpack.yml` with the following content to the directory `bad-overflow-queries`.or execute `codeql pack init cpp/bad-overflow-queries -d` under the directory `cpp` .

   ```yaml
   name: bad-overflow-queries
   version: 0.0.1
   dependencies:
    "codeql/cpp-all" : "0.5.2"
   ```

1. Run the command `CodeQL: Install Pack Dependencies` using the command pallette (View -> Command Pallette...), select the `bad-overflow-queries` pack and click ok.
1. Create a new file, name it `BadOverflowCheck.ql`, save it under `bad-overflow-queries`.

## Documentation links

If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:

- [Learning CodeQL](https://codeql.github.com/docs/writing-codeql-queries/)
- [Learning CodeQL for C/C++](https://codeql.github.com/docs/codeql-language-guides/codeql-for-cpp/)
- [Using the CodeQL extension for VS Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)

## Workshop

The workshop is split into several steps. You can write one query per step, or work with a single query that you refine at each step. Each step has a **hint** that describe useful classes and predicates in the CodeQL standard libraries for C/C++. You can explore these in your IDE using the autocomplete suggestions and jump-to-definition command.

1. Find all relational comparison operations (i.e. `<`, `>`, `<=` and `>=`) in the program.
    <details>
    <summary>Hint</summary>

    Relational comparison are represented by the `RelationalOperation` class in the CodeQL C++ library.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from RelationalOperation cmp
    select cmp
    ```

    </details>

1. Identify relational comparison operations which have an add expression (`+`) as one of the "operands" (arguments).
    <details>
    <summary>Hint</summary>

     - A `+` is represented by the class `AddExpr`
     - `RelationalOperation` has a predicate `getAnOperand()` for getting the operands of the operation.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from AddExpr a, RelationalOperation cmp
    where
      cmp.getAnOperand() = a
    select cmp, a
    ```

    </details>

1. Filter your results to find only cases where there is a variable which is accessed both as an operand of the add expression and the relational operation. For example, the variable `v` in `v + b < v`.
    <details>
    <summary>Hint</summary>

     - The class `Variable` represents variables in the program.
     - `Variable.getAnAccess()` to get an access of the variable.
     - `AddExpr.getAnOperand()` to get an operand of a `+`.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from AddExpr a, Variable v, RelationalOperation cmp
    where
      cmp.getAnOperand() = a and
      cmp.getAnOperand() = v.getAnAccess() and
      a.getAnOperand() = v.getAnAccess()
    select cmp, "Overflow check"
    ```

    </details>

1. The results identified in the previous step are likely to be overflow checks. However, we only want to find "bad" overflow checks. As a reminder, a bad overflow check is one where the type of both of the arguments of the addition (`v + b`) have a type smaller than 32-bits. Write a helper _predicate_ `isSmall` to identify expressions in the program which have a type smaller than 32-bits (4 bytes).
    <details>
    <summary>Hint</summary>

     - The class `Expr`, and the predicate `Expr.getType()` to get the type of each expression.
     - The class `Type` and the predicate `Type.getSize()` to get the size of that type.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    predicate isSmall(Expr e) {
      e.getType().getSize() < 4
    }
    ```

    </details>

1. Refine the query to identify only the additions where both the operands are small, using the `isSmall` predicate you previously wrote.
    <details>
    <summary>Hint</summary>

    - Simple approach: `AddExpr.getLeftOperand()`, `AddExpr.getRightOperand()`
    - Advanced approach: `forall`, `AddExpr.getAnOperand()`
    </details>
    <details>
    <summary>Solution</summary>

    The simplest approach is to add two conditions, one for each operand of the add expression.

    ```ql
    from AddExpr a, Variable v, RelationalOperation cmp
    where
      cmp.getAnOperand() = a and
      cmp.getAnOperand() = v.getAnAccess() and
      a.getAnOperand() = v.getAnAccess() and
      isSmall(a.getLeftOperand()) and
      isSmall(a.getRightOperand())
    select cmp, "Overflow check"
    ```

    This works, but it does require us to refer to `isSmall` twice. Wouldn't it be nice if we could specify the _both operands_ aspect without needing the duplication? Well, we can! QL provides a `forall` _quantifier_. This is specified in three parts:
    - one or more variable declarations
    - a "range", that describes some conditions on the variables
    - a formula that must hold for every value in the range

    For example, we can use this to specify the "both operands are small" condition by saying `forall(Expr op | op = a.getAnOperand() | isSmall(op))`.
    </details>

1. Review the results. You should notice that some include an explicit cast of the result of the addition to `(USHORT)`. This makes these conversions "safe", as it will force them to overflow. Update the query to take account of these "explicit" conversions.
    <details>
    <summary>Hint</summary>

    `Expr.getExplicitlyConverted()`
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from AddExpr a, Variable v, RelationalOperation cmp
    where
      cmp.getAnOperand() = a and
      cmp.getAnOperand() = v.getAnAccess() and
      a.getAnOperand() = v.getAnAccess() and
      forall(Expr op | op = a.getAnOperand() | isSmall(op)) and
      not isSmall(a.getExplicitlyConverted())
    select cmp, "Overflow check"
    ```

    </details>

You should now have exactly one result, which was a [genuine security bug](https://github.com/Microsoft/ChakraCore/commit/2500e1cdc12cb35af73d5c8c9b85656aba6bab4d) in ChakraCore.
