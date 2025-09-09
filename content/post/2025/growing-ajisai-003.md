---
title: "Ajisai コンパイラ育成日誌 第3日 字句解析"
url: "post/growing-ajisai-003"
date: 2025-09-09T10:01:34+09:00
draft: false
description: "自作言語 Ajisai の字句解析器を作っていきます。"
categories:
  - "技術"
tags:
  - "自作プログラミング言語"
  - "言語処理系自作"
  - "Ajisai"
share: true
---

（今回のコード: [day001-010/day003](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day003) ）

[前回](/post/growing-ajisai-002)はファイルからソースコードを読めるようにしたのでした。ただ、普通のコンパイラがやるような解析や変換はまだ何も行なっていません。そこで、今回は字句解析器を作っていきます。

とりあえず字句（トークン）の型を考えてみます：

```ts
type Span = {
  srcPath: string;
  srcContent: string;
  start: number;
  end: number;
};

type OpToken = {
  tag: "+" | "-" | "*" | "/" | "%";
};
type ParenToken = {
  tag: "(" | ")";
};
type IntToken = {
  tag: "integer";
  value: string;
};
type Token = (IntToken | OpToken | ParenToken) & { span: Span };
```

`Span` 型は、後でエラー表示をするときに、エラーが起きた箇所のソースコード中の位置を表示できるように、関係する情報をまとめた型です。各種トークンが共通して持っているフィールド `span` に格納されます。

自分にとっての分かりやすさのために、`Token` 型を `OpToken`, `ParenToken`, `IntToken` の3種類に分類して書きました（分けて書く必要はないのですが）。この中では `IntToken` だけ特殊で、字句解析器で読み取った整数リテラルを格納する `value` フィールドがあります。

さて、字句解析器の役割は次の `Lexer` クラスに担わせます：

```ts
class Lexer {
  #curPos = 0;
  // ...

  constructor(
    public readonly srcPath: string,
    public readonly srcContent: string,
  ) {}

  // ...
}
```

コンストラクタ引数に `srcPath` と `srcContent` を渡しています。ということで、今のところこの `Lexer` クラスは Ajisai 言語のソースファイルごとにインスタンスを作る予定です。また、`#curPos` は `srcContent` 中で次に読む文字を示すインデックスです。

`srcContent` 中の文字を一つ読み進めるには `nextChar()` メソッドを使います。一文字読む度に `#curPos` をインクリメントします：

```ts
class Lexer {
  // ...

  private nextChar(): string | undefined {
    if (this.#curPos === this.srcContent.length) return undefined;
    const ch = this.srcContent[this.#curPos]!;
    this.#curPos++;
    return ch;
  }

  // ...
}
```

しかし手書きの構文解析器を作ったことがある人は、一文字読み進めるのではなく「次の一文字をチラ見する（`#curPos` はインクリメントしない）」という操作も欲しいと思うはずです。実際、後で使い所が出てきます。それを行う `peekChar()` メソッドは次の通りです：

```ts
class Lexer {
  // ...

  private peekChar(): string | undefined {
    if (this.#curPos === this.srcContent.length) return undefined;
    return this.srcContent[this.#curPos]!;
  }

  // ...
}
```

そして、実際に文字列を読み取った結果からトークンを作って返すメソッド `nextTokenImpl()` は次の通りです（実装の都合で、これにもう少し別の処理を加えたものを `nextToken()` メソッドとして公開しています）：

```ts
class Lexer {
  // ...

  private nextTokenImpl(): Token | SyntaxError {
    let startPos = this.#curPos;

    while (true) {
      const ch = this.nextChar();
      if (!ch) return new SyntaxError(this.newSpan(startPos, this.#curPos), { tag: "reachEOF" });

      // 空白文字をスキップ
      if (ch === " " || ch === "\t" || ch === "\n" || ch === "\r") {
        startPos++;
        continue;
      }

      // 数値リテラル
      if (this.isDigit(ch)) return this.readNumber(ch, startPos);

      // 記号始まりの字句
      if (this.isPunct(ch)) return this.readPunct(ch, startPos);

      return new SyntaxError(this.newSpan(startPos, this.#curPos), { tag: "invalidChar" });
    }
  }

  // ...
}
```

空白文字は読み飛ばしたいので、whileループでまず最初に一文字読み進めて、空白文字だったらcontinueします。それ以外の文字だったら、来た文字の種類にに応じて別々の解析メソッドを実行します。数字から始まっている文字列は明らかに数値リテラルなので `readNumber()` メソッドで解析して、その結果（数値リテラルのトークン）を返しています。今のところ、それ以外のトークンは記号のみなので、`isPunct()` メソッドで適切な記号から始まっていることを確認してから `readPunct()` メソッドで解析して、その結果を返します（ただし、数値リテラルは `-` や `+` から始まることもある予定です。とはいえ、今回のアプローチでは、字句解析の時点で `-` と `+` が単項演算子なのか二項演算子なのか区別がつきません。次回に実装する構文解析ステップでその辺りの解析をやっていきたいです）。

`readNumber()` メソッドは、今後浮動小数点数も扱うときには浮動小数点数リテラルも読むことになると思いますが、今のところは単純な整数リテラルのみを読むことにしたいと思います。「単純」とはどういうことかというと、以下の実装を見ると分かるように、まだ10進数表記にしか対応していません。それから、このメソッドの実装に先ほど載せた `peekChar()` メソッドが使われています。数値リテラルが終わった後の次の文字まで `nextChar()` メソッドで読み進めてしまうと、次に `nextTokenImpl()` メソッドで読み始める位置が本来読み始めたい位置より１つ後ろになってしまいます。なので、`#curPos` を進めないで文字をチラ見する `peekChar()` メソッドが必要になるのです：

```ts
class Lexer {
  // ...

  private readNumber(initial: string, startPos: number): Token | SyntaxError {
    let value = initial;
    let next = this.peekChar();

    // TODO: ２進数、１６進数等の数値リテラル
    // 実装するまでは、二桁以上の整数リテラルで先頭が 0 のケースは禁止する
    if (initial === "0" && next && this.isDigit(next))
      return new SyntaxError(this.newSpan(startPos, this.#curPos + 1), { tag: "invalidNumLit"});

    while (next && this.isDigit(next)) {
      this.nextChar();
      value += next;
      next = this.peekChar();
    }

    return { tag: "integer", value, span: this.newSpan(startPos, this.#curPos) };
  }

  // ...
}
```

ところで、今回のコンパイラ実装にあたっては、エラー伝播は独自のエラークラスを定義した上で、関数の戻り値の型として、成功時に返す値の型とそのエラークラスのユニオン型を用いるという方法を取っています。ここまでの例では `nextTokenImpl()` と `readNumber()` において、`SyntaxError` クラスを定義した上で、`Token | SyntaxError` を戻り値にしています[^error-propagation]。呼び出し側では `instanceof` で戻り値がエラーかどうかを判定できます。

`SyntaxError` クラスは以下のような感じです：

```ts
type SyntaxErrorInfo =
  { tag: "reachEOF" }
| { tag: "invalidChar" }
| { tag: "invalidNumLit" }
| { tag: "unreachable" };

class SyntaxError {
  constructor(
    public readonly span: Span,
    public readonly errorInfo: SyntaxErrorInfo,
  ) {}

  private message1(): string {
    switch (this.errorInfo.tag) {
      case "reachEOF":
        return "reach EOF";
      case "invalidChar":
        return "invalid character";
      case "invalidNumLit":
        return "invalid number literal";
      case "unreachable":
        return "unreachable (maybe compiler's bug)";
    }
  }

  message(): string {
    return `${spanToString(this.span)}\nsyntax error: ${this.message1()}`;
  }
}
```

`message()` メソッドでエラーメッセージを作れるようになっています。自作言語処理系を作るとき、いつもエラー表示をサボっていたのですが、今回はちょっとはしっかりしようと思って実装しました。エラー表示で重要な役割を果たしている `spanToString()` 関数についても紹介したかったのですが、記事が長くなりそうなので別の機会にしようと思います……。

最後に、`nextToken()` メソッド、`peekToken()` メソッドを紹介します。字句解析器内では `nextChar()`、`peekChar()` があると便利だったように、次回以降に取り組む構文解析においては、ソースコードを読み進めてトークンを得る `nextToken()` メソッドと、読み進めずに次のトークンをチラ見する `peekToken()` メソッドがあると便利なので、用意しておきました：

```ts
class Lexer {
  // ...
  #tokenBuf: Token | SyntaxError | undefined = undefined;

  // ...

  private prepareTokenForPeek() {
    this.#tokenBuf = this.nextTokenImpl();
  }

  nextToken(): Token | SyntaxError {
    const curToken = this.#tokenBuf ?? this.nextTokenImpl();
    this.prepareTokenForPeek();
    return curToken;
  }

  peekToken(): Token | SyntaxError {
    if (!this.#tokenBuf) this.prepareTokenForPeek();
    return this.#tokenBuf!;
  }
}
```

考え方としては、「チラ見用のトークンを置いておくバッファを用意して、常にそこにチラ見用のトークンがあるようにする」という感じです。ただし、`nextToken()` を最初に呼び出す場合は `#tokenBuf` が空のはずなので `nextTokenImpl()` を直接呼んで戻り値のトークンを用意する必要があります。`peekToken()` を最初に呼び出す場合も、一度 `#tokenBuf` をトークンで埋めてから返すようにする必要があります。

実装の全ては紹介しきれていませんが、大筋は追えたと思います。今回実装した字句解析器を実際に使ってみましょう。まだ構文解析以降のフェーズ、特にコード生成のフェーズを実装できていないので、今回は単に得られた各トークンを再び文字列の表現に戻して結合してからC言語のソースコードに埋め込む、というやり方でやってみます：

```ts
function buildMain(_options: Options, ...args: Arguments) {
  // ...

  // Ajisai 言語のソースコードをファイルから読み込む
  const ajisaiSource = Deno.readTextFileSync(sourceFilePath);

  // とりあえず字句解析で得たトークン列からソースコードを再構築
  const lexer = new Lexer(sourceFilePath, ajisaiSource);
  let lexedSource = "";
  while (true) {
    const result = lexer.nextToken();
    if (result instanceof SyntaxError) {
      if (result.errorInfo.tag === "reachEOF") break;
      console.log(result.message());
      Deno.exit(1);
    }
    switch (result.tag) {
      case "integer":
        lexedSource += result.value;
        break;
      default:
        lexedSource += result.tag;
        break;
    }
  }

  // C 言語のソースコード
  const cSource = `#include <stdio.h>

int main() {
  printf("%d\\n", ${lexedSource});
  return 0;
}
`;

  // ...
}
```

試しに前回用意した `examples/answer.ajs` や `examples/arith.ajs` をコンパイルしてみたところ、うまくいきました。

前回よりも賢くなった面も見てみましょう。今回は、まだ識別子（変数名等に使うトークン）の字句解析には対応していないので、識別子を書くと構文エラーになります。`(123 + abc)` という内容のソースファイル `hoge.ajs` をコンパイルしてみて確かめてみましょう：

```
> deno -A ajisai.ts build hoge.ajs
hoge.ajs:1:8-1:8:
(123 + abc)
       ^
syntax error: invalid character
```

コンパイラは文字 `a` が読めなくてエラーを返しました。

また、今回はまだ二桁以上かつ先頭が0の整数はエラーを報告するようにしています（C言語的にはそのような整数は8進数なのですが、整数リテラルの字句解析の複雑さが増すので今回は10進数以外には対応しませんでした）。`hoge.ajs` の内容を `(123 + 0456)` にして確かめてみましょう：

```
> deno -A ajisai.ts build hoge.ajs
hoge.ajs:1:8-1:9:
(123 + 0456)
       ^
syntax error: invalid number literal
```

期待通りです。まだまだできることの少ないコンパイラですが、少し成長したのではないでしょうか。

次回は構文解析以降に取り組んでみたいと思います。

（今回のコード: [day001-010/day003](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day003) ）

[^error-propagation]: この方法は「[TypeScriptのエラーハンドリングまとめ](https://zenn.dev/motojouya/articles/typescript_error_handling#union-return-style)」という記事で Union Return Style という名前で紹介されていて、便利そうだったので採用しました。Ajisai 言語自体もエラー伝播の仕組みとして例外を実装せず、Rust のように `Result` 型をたらい回しさせる予定なので、今のうちに近い感覚の方法で実装したいというのもありました。
