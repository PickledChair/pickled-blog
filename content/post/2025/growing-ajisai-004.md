---
title: "Ajisai コンパイラ育成日誌 第4日 構文解析"
url: "post/growing-ajisai-004"
date: 2025-09-26T20:22:39+09:00
draft: false
description: "自作言語 Ajisai の構文解析器を作っていきます。"
categories:
  - "技術"
tags:
  - "自作プログラミング言語"
  - "言語処理系自作"
  - "Ajisai"
share: true
---

（今回のコード: [day001-010/day004](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day004) ）

[前回](/post/growing-ajisai-003)は字句解析器（の原型）を作りました。今回は構文解析以降のステップを作っていこうと思います。

その前に、コードの見通しが悪くなっていきそうなので、ファイル分割をすることにします。分割後のディレクトリ構成は以下の通りです（`parser` ディレクトリ以下の定義は `parser/mod.ts` で export しています）：

```
$ tree
.
├── ajisai.ts
├── examples
│   ├── answer.ajs
│   └── arith.ajs
└── parser
    ├── error.ts
    ├── lexer.ts
    ├── mod.ts
    ├── span.ts
    └── token.ts
```

だいたいどのファイルに何のコードが置いてあるかは想像がつくと思います。今回はまず、`parser` ディレクトリ内に `parser/ast.ts` と `parser/parser.ts` を追加し、そこに構文解析を担うコードを書いていきます。

手始めに、`parser/ast.ts` に抽象構文木（AST）の型を定義します（まだ二項演算と単項演算、整数リテラルしか定義するものがありませんが）：

```ts
import { Span } from "./span.ts";

export type Expr =
(
  {
    tag: "binaryExpr";
    left: Expr;
    op: "+" | "-" | "*" | "/" | "%";
    right: Expr;
  }
| {
    tag: "unaryExpr";
    op: "+" | "-";
    operand: Expr;
  }
| {
    tag: "intLit";
    value: string;
  }
) & { span: Span };
```

字句解析器とASTの型が揃い、構文解析器を作る準備が整いました。素朴な再帰下降パーサーを書いていきます。

字句解析器のよくある利用パターンに関して、ヘルパーメソッドの `eat()` と `expect()` を定義しました：

- `eat()` : トークンをチラ見してみて、期待する種類のトークンならそのトークンを、そうでなかったら `undefined` を返します。`undefined` を返した場合、呼び出し元では構文解析をやめずに、他の種類のトークンを試し続ける想定のメソッドです。
- `expect()` : トークンをチラ見してみて、期待する種類のトークンならそのトークンを、そうでなかったら `SyntaxError` を返します。`SyntaxError` を返した場合、呼び出し元では構文解析を中断し、エラーをさらに上位の呼び出し元へ伝播させていく想定のメソッドです。

```ts
import { Lexer } from "./lexer.ts";
import { SyntaxError } from "./error.ts";
import { Token } from "./token.ts";

type TokenTag = Token["tag"];

export class Parser {
  #lexer: Lexer;

  constructor(srcPath: string, srcContent: string) {
    this.#lexer = new Lexer(srcPath, srcContent);
  }

  // ...

  private eat(tag: TokenTag): Token | undefined {
    const token = this.#lexer.peekToken();
    if (token instanceof SyntaxError) return undefined;

    if (token.tag === tag) {
      this.#lexer.nextToken();
      return token;
    } else {
      return undefined;
    }
  }

  private expect(tag: TokenTag): Token | SyntaxError {
    const token = this.#lexer.peekToken();
    if (token instanceof SyntaxError) return token;

    if (token.tag === tag) {
      this.#lexer.nextToken();
      return token;
    } else {
      return new SyntaxError(
        token.span,
        { tag: "unexpectedToken", expected: tag, got: token },
      );
    }
  }
}
```

（と、せっかく定義したはいいものの、使ったり使わなかったりします。割と適当に実装しています……。）

今回実装している構文解析器は、今のところ最終的に単一の式のASTを返すことになります。それを返すトップのメソッド `parseExpr()` を外部に公開します。そこから、構文規則に従って、下の階層の規則へと構文解析を進めていくのですが、まず現時点で最も下の階層の規則 primary を解釈する `parsePrimary()` メソッドを下に示します：

```ts
import { Expr } from "./ast.ts";
import { SyntaxError } from "./error.ts";

export class Parser {
  // ...

  // expr = ???
  parseExpr(): Expr | SyntaxError {
    // return ???;
  }

  // ...

  // primary = ("(" expr ")") | integer
  private parsePrimary(): Expr | SyntaxError {
    if (this.eat("(")) {
      const expr = this.parseExpr();
      if (expr instanceof SyntaxError) return expr;
      const rParen = this.expect(")");
      if (rParen instanceof SyntaxError) return rParen;
      return expr;
    }

    const int = this.expect("integer");
    if (int instanceof SyntaxError) return int;
    switch (int.tag) {
      case "integer":
        return {
          tag: "intLit",
          value: int.value,
          span: int.span,
        };
      default:
        return new SyntaxError(int.span, { tag: "unreachable" });
    }
  }

  // ...
}
```

`parsePrimary()` メソッドでは、次のトークンが左の丸括弧 `(` だった場合、`(1 + 2)` のようにグループを作る式だと解釈します。ここで、`parseExpr()` メソッドを呼び出してもう一度上の階層の規則から解釈を始めます。下の階層から上の階層へと再帰するような文法を考えるのは、お決まりのやり方とは言え楽しい部分です。`this.expect(")")` で右の括弧をチェックするのも忘れずに。

もし `(` で始まらない場合は、整数リテラルが来るはずなのでそれを解釈します。現時点ではこれらが規則 primary の全パターンです。

さて、primary の１つ上の規則 unary を解釈する `parseUnary()` メソッドは以下の通りです：

```ts
import { Expr } from "./ast.ts";
import { SyntaxError } from "./error.ts";
import { mergeSpans } from "./span.ts";

export class Parser {
  // ...

  // unary = ("+" | "-")* primary
  private parseUnary(): Expr | SyntaxError {
    const opToken = this.#lexer.peekToken();
    if (opToken instanceof SyntaxError) return opToken;

    switch (opToken.tag) {
      case "+":
      case "-": {
        this.#lexer.nextToken();

        const operand = this.parseUnary();
        if (operand instanceof SyntaxError) {
          if (operand.errorInfo.tag === "reachEOF")
            return new SyntaxError(opToken.span, { tag: "unaryOperandNotFound" });
          else
            return operand;
        }

        return {
          tag: "unaryExpr",
          op: opToken.tag,
          operand,
          span: mergeSpans(opToken.span, operand.span)!,
        };
      }
      default:
        break;
    }

    return this.parsePrimary();
  }

  // ...
}
```

数値の単項演算子 `+` と `-` を読み続けられる限り、`parseUnary()` メソッドを再帰させ続けて読み進めます。もしそれらの演算子が来なかったら、最後の行の `return this.parsePrimary();` に到達して、構文規則 primary の解釈を試みます。

ここで、解釈し終わった unary ノード全体のソースコード上の位置は、演算子とオペランドの位置を融合させたものになります。それを計算するために、前回のうちに（実は）定義していた `mergeSpans()` 関数を用いています。

残りの構文規則 mul と add は似通っているので、代表して規則 add の解釈をおこなう `parseAdd()` メソッドの実装を示します。なお、現在は規則 add が最も上の階層の規則なので、外部に公開する `parseExpr()` メソッドでは、単に `parseAdd()` メソッドを呼び出した結果を返します：

```ts
import { Expr } from "./ast.ts";
import { SyntaxError } from "./error.ts";
import { mergeSpans } from "./span.ts";

export class Parser {
  // ...

  // expr = add
  parseExpr(): Expr | SyntaxError {
    return this.parseAdd();
  }

  // add = mul (("+" | "-") mul)*
  private parseAdd(): Expr | SyntaxError {
    let left = this.parseMul();
    if (left instanceof SyntaxError) return left;

    while (true) {
      const opToken = this.#lexer.peekToken();
      if (opToken instanceof SyntaxError) {
        if (opToken.errorInfo.tag === "reachEOF")
          return left;
        else
          return opToken;
      }

      let op: "+" | "-";
      switch (opToken.tag) {
        case "+":
        case "-":
          op = opToken.tag;
          break;
        default:
          return left;
      }
      this.#lexer.nextToken();

      const right = this.parseMul();
      if (right instanceof SyntaxError) {
        if (right.errorInfo.tag === "reachEOF")
          return new SyntaxError(opToken.span, { tag: "rhsNotFound" });
        else
          return right;
      };
      left = {
        tag: "binaryExpr",
        left, op, right,
        span: mergeSpans(left.span, right.span)!,
      };
    }
  }

  // ...
}
```

このように、二項演算のノードを組み立てるのに「左辺・演算子・右辺」を一組解釈し終えたら、それをいったん丸ごと左辺のノードのための変数 `left` に代入し、そのままループして可能な限り次の「演算子・右辺」の解釈を続けるという実装が、よくあるお決まりのパターンだと思います。

規則 add の解釈では左辺と右辺の式がそれぞれ mul、規則 mul の解釈では左辺と右辺の式がそれぞれ unary、といった階層構造を作ることで、`+` や `*` の演算子の優先順位を表現することができています。ここで、`parseAdd()` と `parseMul()` の実装があまりにも似通っているので、全くもってDRYではないのですがご容赦ください（これを解消するには Pratt パーサー[^pratt-parser]のテクニックを使うのが良いと思いましたが、今回は気が向かなかったのでそこまで実装しませんでした）。

ここまで構文解析器の実装を見てきて気づいたかもしれませんが、字句解析器と構文解析器で同じエラー型 `SyntaxError` クラスを使っています。`SyntaxError` クラスは初期化時にエラーの詳細情報の型 `SyntaxErrorInfo` を引数に渡しますが、前回はまだ字句解析器向けのエラー情報しか定義していなかったので、今回は構文解析器向けのエラー情報も追加しました（以下には示していませんが、対応するエラーメッセージも追加しました）：

```ts
type TokenTag = Token["tag"];

export type SyntaxErrorInfo =
// for Lexer
  { tag: "reachEOF" }
| { tag: "invalidChar" }
| { tag: "invalidNumLit" }
// for Parser
| { tag: "unexpectedToken", expected: TokenTag, got: Token }
| { tag: "rhsNotFound" }
| { tag: "unaryOperandNotFound" }
| { tag: "unreachable" };
```

これで構文解析器の（最初の）実装は終わりです。構文解析については今回実装した `Parser` クラスのインスタンスを生成して `parseExpr()` メソッドを実行すれば良いだけですが、それらを簡単に行えるように `parse()` 関数にまとめておきました：

```ts
export function parse(srcPath: string, srcContent: string): Expr | SyntaxError {
  const parser = new Parser(srcPath, srcContent);
  return parser.parseExpr();
}
```

構文解析器が済んだので、次は意味解析……と行きたいところですが、気力が続かなかったので、そのステップはすっ飛ばして、今回は AST からいきなりコード生成を行います。今までは字句解析器で得られたトークンを得られた順に再びくっつけていっただけでしたが、トークン列とは違って AST は再帰構造をしているので、再帰関数でC言語の式を組み立てる必要があります。それを行なっているのが、とりあえず新しく追加した `codegen/codegen.ts` のコードです：

```ts
import { Expr } from "../parser/mod.ts";

function makeCExpr(expr: Expr): string {
  switch (expr.tag) {
    case "binaryExpr":
      return `(${makeCExpr(expr.left)} ${expr.op} ${makeCExpr(expr.right)})`;
    case "unaryExpr":
      if (expr.op === "+") {
        return makeCExpr(expr.operand);
      } else {
        return `(${expr.op}${makeCExpr(expr.operand)})`;
      }
    case "intLit":
      return `${expr.value}`;
  }
}

export function codegen(ast: Expr, filePath: string) {
  // C 言語のソースコード
  const cSource = `#include <stdio.h>

int main() {
  printf("%d\\n", ${makeCExpr(ast)});
  return 0;
}
`;

  // filePath に C ソースコードを書き込み
  try {
    const file = Deno.openSync(
      filePath,
      { write: true, create: true, truncate: true }
    );
    const encoder = new TextEncoder();
    file.writeSync(encoder.encode(cSource));
  } catch {
    console.error(`couldn't write C source to "${filePath}"`);
    Deno.exit(1);
  }
}
```

`makeCExpr()` 関数が再帰しつつC言語の式を組み立てています。C言語側の演算子の優先順位を考慮しなくても済むように、出来上がった式が括弧まみれになるのはご愛嬌、ということで……。

最後に、ここまでに実装した `parse()` 関数と `codegen()` 関数を `buildMain()` 関数中で用いれば、再び Ajisai 言語のコンパイラが動くようになります：

```ts
// ...

import { parse, SyntaxError } from "./parser/mod.ts";
import { codegen } from "./codegen/mod.ts";

// ...

function buildMain(_options: Options, ...args: Arguments) {
  // ...

  // Ajisai 言語のソースコードをファイルから読み込む
  const ajisaiSource = Deno.readTextFileSync(sourceFilePath);

  // ソースコードを構文解析し、ASTを構築
  const ast = parse(sourceFilePath, ajisaiSource);
  if (ast instanceof SyntaxError) {
    console.log(ast.message());
    Deno.exit(1);
  }

  // 出力ディレクトリ ajisai-out を準備
  const outputDirName = "ajisai-out";
  // （省略）

  // ajisai-out/main.c に C ソースコードを書き込み
  const outputCFilePath = `${outputDirName}/main.c`;
  codegen(ast, outputCFilePath);

  // ...
}
```

色々実装しましたが、振り返ってみると、せっかくソースコードを変換して得た抽象構文木を、また似たようなC言語のソースコードに戻すだけの無意味な処理を書いたように感じます。一応 `examples` ディレクトリ以下のコードなどを期待通りにコンパイルできるものの、残念ながらまだそこまで面白みを感じませんね。

それでも、前回よりも改善した部分はあります。例えば構文的に不完全なソースコードを書いた場合、前回までの実装では Ajisai コンパイラを貫通して不完全なコードがCコンパイラに渡り、Cコンパイラが構文エラーを表示します：

```
$ echo "1 +" > hoge.ajs
$ deno -A ajisai.ts build hoge.ajs
ajisai-out/main.c:4:20: error: expected expression
    4 |   printf("%d\n", 1+);
      |                    ^
1 error generated.
```

それが今回の実装で、Ajisai コンパイラ自体で構文エラーを検知することができるようになりました：

```
$ echo "1 +" > hoge.ajs
$ deno -A ajisai.ts build hoge.ajs
hoge.ajs:1:3-1:3:
1 +
  ^
syntax error: binary expression has no valid right-hand operand
```

しかし、上記のような構文エラーは検知できても、ソースコードの意味的な部分のエラーはまだ検知できません。例えば、現時点では整数値はたまたまC言語の int 型の範囲のみを扱える状態になっています。それを超える範囲の整数値を書くと、やはり Ajisai コンパイラを貫通してCコンパイラで警告が出ます：

```
$ echo 2147483648 > hoge.ajs
$ deno -A ajisai.ts build hoge.ajs
ajisai-out/main.c:4:18: warning: format specifies type 'int' but the argument has type 'long' [-Wformat]
    4 |   printf("%d\n", 2147483648);
      |           ~~     ^~~~~~~~~~
      |           %ld
1 warning generated.
```

あまり真面目でない趣味のコンパイラ実装なので、これをこのまま放っておくというのも手なのですが、せっかくなのでこのエラーも Ajisai コンパイラが処理するようにしていきたいです。それは次回、意味解析ステップの実装を通して行いたいと思います。また、気が向いたら構文解析や意味解析のテストも追加できればなあ、と考えています。

（今回のコード: [day001-010/day004](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day004) ）

[^pratt-parser]: Pratt パーサーは、再帰下降による構文解析中に、演算子の優先順位を考慮しつつ、出会った演算子に対応する解析関数を選んで構文解析していけるようにする手法のパーサー。「[Go言語でつくるインタプリタ](https://www.oreilly.co.jp/books/9784873118222/)」や「[インタプリタの作り方 (CRAFTING INTERPRETERS)](https://book.impress.co.jp/books/1122101087)」にも登場する。また、Rust の PEG パーサージェネレータ [pest](https://pest.rs/) にも Pratt パーサーを定義する機能がついていたりする。
