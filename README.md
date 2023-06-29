# イントロダクション
本レポジトリは、CodeQL学習向け教材となります。特定の問題に対する解決策を示すことで、より一般的な問題に対して参考になることを期待します。

本ワークショップは、プログラミング言語ごとに分かれています。現状、C++、charp、Java、JavaScriptをサポートします。

ディレクトリの構造は、次のようになっています。

    language/project/content

ただし、いくつかの古いワークショップが含まれており、それらについては、以下のようなディレクトリ構造となります。

    language/project


これらプロジェクトの難しさのレベルは、それぞれです。さらに、いくつかは、既存データベースを使ってCodeQLをカバーします。他のものについては、コマンドライン上で新たにコードからデータベースを生成するものも含まれています。

簡単に、それぞれの言語別プロジェクトのディレクトリについて説明します。
```
cpp
├── codeql-dataflow-sql-injection
├── codeql-workshop-cpp-bad-overflow-check.md
└── introduction
    ├── codeql-workshop-for-cpp.md
    ├── session-1
    ├── session-2
    ├── session-3
    └── session-4

csharp
├── codeql-workshop-csharp-unsafe-pointer-arithmetic.md
├── codeql-workshop-csharp-zipslip.md
└── top-down-vulnerability-guide.md

go
├── codeql-go-sqli
├── codeql-workshop-go-bad-redirect-check.md
├── oauth2-notes.org


java
├── Introduction\ to\ CodeQL\ -\ Java.pdf           | 説明資料 
├── codeql-java-workshop-notes.md                   | ワークショップのノート 
├── apache-struts-online.txt                        |
├── codeql-dataflow-sql-injection/                  | 必要なファイルがこのディレクトリ配下にあります。 
|                                                   | データベース構築ツール、ソースビルド (初心者向け)
├── codeql-java-workshop-sqlinjection.md            | sqlインジェクション OWASP Security Shepherd
├── java-unsafe-deserialization.md                  | レクチャノート
├── unsafe-deserialization-apache-struts.md         | 危険なデシリアライゼーション, 圧縮, データベースビルド
└── workshop-java-mismatched-loop-condition.md      |

javascript
├── codeql-js-goof-workshop  | 必要な全ての例がこのディレクトリにあります。手順、データベースビルド、ソースビルド(初心者向け)
├── codeql-workshop-javascript-unsafe-jquery-calls.md | pure codeql, beginner 

python
└── codeql-dataflow-sql-injection | Full example, beginner, db build, source build
```

# Status & Roadmap
These are actively developed and used workshops and are subject to editorial changes at any time.  We are planning to add
intermediate and advanced material as time permits.

# Setup and running
Currently all projects require installing VS Code and the CodeQL extension.  They can
be run on linux, macOS, and Windows.  Some additionally require the CodeQL command
line tools.  See the individual project's instructions, or 
[here for the cli](./common/cli-for-codeql.org) and 
[here for VS Code](./common/vscode-for-codeql.org)

# Contributing
New tutorials should use the `language/project/content` structure to allow for
expansion.  

This is a **staging** area, so the rules are relaxed:
- If you have bare content that you have used, it's good enough.
- If you have a writeup that you think you will use, it's good enough.
- Err on the side of too little content; maybe someone else will use it as a starting point.
- Don't wait for PR's when you're adding new content, or making minor changes.

While evolving content, the goal should be learning and explaining CodeQL and
the content should eventually cover these items:
1. A high-level problem description
2. The specific parts of the original source code to be analyzed 
3. Descriptions of the CodeQL predicates/classes developed
4. A description of the final query
     




