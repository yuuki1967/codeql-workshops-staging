# CodeQL Workshop for JavaScript : Improper verification of cryptographic signature in tEnvoy

- Analyzed language: JavaScript
- Difficulty level: 1/3

## Overview

 - [Problem statement](#problemstatement)
 - [Workshop](#workshop)
   - [Section 1: Finding Sources](#section1)
   - [Section 2: Finding Sinks](#section2)
   - [Section 3: Data Flow](#section3)
   - [Section 5: Final Query](#finalquery)
   - [Stretch Exercise](#stretchexercise)
 - [What's next](#whatsnext)
 
## Problem Description  <a id="problemstatement"></a>

Cryptographic failures climbed to the second position in the 2021 OWASP Top 10. Such failures during verification can compromise the confidentiality and integrity of data. In this workshop, we will be looking at [CVE-2021-32685](https://github.com/TogaTech/tEnvoy/security/advisories/GHSA-7r96-8g3x-g36m): Improper verification of a cryptographic signature in the JavaScript encryption library [tEnvoy](https://tenvoy.js.org/) (`< v7.0.3`).

This vulnerability involves a function called [`verify()`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2138) which returns a dict with a field named [`verified`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2150) which in turn holds `true` if the SHA-512 signature stored within parameter [`signed`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2138) is valid. A second function, [`verifyWithMessage()`](https://github.com/TogaTech/tEnvoy/blob/4e7169cfa1107077a2d55eac8b03f9fce299783e/node/tenvoy.js#L2169), uses `verify()` to check the validity of a message's signature. Unfortunately, instead of checking the `verified` field, it uses `verify()`'s returned dict. A JavaScript dict interpreted as a boolean will always evaluate to `true`, thus rendering the check ineffectual.

This vulnerability exists for versions < 7.03. Technical fix: [patch commit](https://github.com/TogaTech/tEnvoy/commit/a121b34a45e289d775c62e58841522891dee686b)

To detect this vulnerability in CodeQL, we will need to use data flow. In the program we need to identify the sources and sinks and use the dataflow library to identify if there is flow between them. 

**Source**

In the `verify` function: 
  ``` javascript
     {
		verified: _nacl.sign.detached.verify(hash, signature, this.getPublic(_getPassword())),
		hash: signed.split("::")[0]
	 };
   ```
**Sink**

In the `verifyWithMessage` function:

`this.verify(signed, password)`

## Workshop  <a id="workshop"></a>

The workshop is split into several steps. You can write one query per step, or work with a single query that you refine at each step.

Each step has a hint that describe useful classes and predicates in the CodeQL standard libraries for JavaScript and keywords in CodeQL. You can explore these in your IDE using the autocomplete suggestions and jump-to-definition command.

Each step has a Solution that indicates one possible answer. Note that all queries will need to begin with import `javascript`, but for simplicity this may be omitted below.

### Section 1: Finding Sources  <a id="section1"></a>


1. Find all functions
    <details>
    <summary>Hint</summary>

    A function is called a `Function` in the CodeQL JavaScript library.
    </details>
     <details>
    <summary>Solution</summary>
    
    ```CodeQL
    from Function f
    select f
    ```
    </details>
2. Find all functions named `verify`
    
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


3. In the previous step, there were 94 functions named "verify". Not all of those trigger the vulnerability we are after, so we need to be a bit more specific. Let's find only those functions whose body contains an object expression with a property named `verified`. CodeQL ships with support for various syntactic JavaScript constructs. We may consult the [documentation](https://codeql.github.com/docs/codeql-language-guides/codeql-library-for-javascript/) to find out how we can reason about [object expressions](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/Expr.qll/type.Expr$ObjectExpr.html). You may also find what you want by using autocompletion which will display some documentation. Try to use the AST viewer to understand the syntactic structure of the program and the `verified` property.

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

### Section 2: Finding Sinks <a id="section2"></a>

1. Recall that the vulnerability arose from the fact that a caller used its return value, an object expression, as if it was a boolean. More specifically, it was being used as if it was a condition variable with a boolean value. CodeQL's JavaScript library has a class which describes expressions which are used as conditions: the [ConditionGuardNode](https://codeql.github.com/codeql-standard-libraries/javascript/semmle/javascript/CFG.qll/type.CFG$ConditionGuardNode.html). This is a type of node in the Control Flow Graph which records that some condition is known to be truthy or falsy at the point in the program where it appears. Let's use it to define a predicate `isCondition()`. Make use of the [exists](https://codeql.github.com/docs/ql-language-reference/formulas/#exists) quantifier.

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

  
## Section 3: Data Flow <a id="section3"></a>

We have now identified (a) functions in the program whose body contains an object expression with a property named `verified` (b) places in the program where conditions are known to be truthy or falsy. We now want to tie these two together to ask: does (a) ever _flow_ to (b)? 

In program analysis we call this a _data flow_ problem. Data flow helps us answer questions like: does this expression ever hold a value that originates from a particular other place in the program?

We can visualize the data flow problem as one of finding paths through a directed graph, where the nodes of the graph are elements in program, and the edges represent the flow of data between those elements. If a path exists, then the data flows between those two nodes.

CodeQL for JavaScript provides data flow analysis as part of the standard library. You can import it using `semmle.code.javascript.dataflow.DataFlow`. The library models nodes using the `DataFlow::Node` CodeQL class. These nodes are separate and distinct from the AST (Abstract Syntax Tree, which represents the basic structure of the program) nodes, to allow for flexibility in how data flow is modeled.

There are a small number of data flow node types – expression nodes and parameter nodes are most common.

In this section we will create a data flow query by populating this template:


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


1. Complete the `isSource` predicate using the query you wrote previously

    <details>
    <summary>Hint</summary>

    - You can translate from a query clause to a predicate by:
       - Converting the variable declarations in the `from` part to the variable declarations of an `exists`
       - Placing the `where` clause conditions (if any) in the body of the exists
       - Adding a condition which equates the `select` to one of the parameters of the predicate.
   
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

2. Complete the `isSink` predicate 
    <details>
    <summary>Hint</summary>
     Complete the same process as above.
   </details>
   <details>
    <summary>Solution</summary>

     ```ql
        override predicate isSink(DataFlow::Node sink) { isCondition(sink.asExpr()) }      
     ```
  </details>


You can now run the completed query. What results did you get? Was it what you were expecting? 

## Final Query <a id="finalquery"></a>
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