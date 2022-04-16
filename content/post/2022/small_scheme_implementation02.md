---
title: "Scheme 処理系自作 (2) 字句解析・構文解析"
url: "post/small_scheme_implementation02"
date: 2022-04-16T00:09:05+09:00
draft: false
description: "学習のための自作Scheme処理系に字句解析と構文解析の処理を追加しました"
toc: true
categories:
  - "技術"
tags:
  - "Scheme"
  - "言語処理系自作"
share: true
---

## 今回の作業

[前回](/post/small_scheme_implementation01)は Scheme オブジェクトを生成・表示するコードを書きました。今回はこれを、単純な Scheme のソースコード文字列から生成できるようにします。

ソースコードから Scheme オブジェクトを生成するために、とりあえず以下の２つのステップを踏むことにします：


- **字句解析 (tokenize)** : ソースコードの文字列をトークン列に変換する
- **構文解析 (parse)** : トークン列を Scheme オブジェクトに変換する

ここで、構文解析の手順で生成するのは通常は抽象構文木であることに注意します。言語固有のオブジェクトはたいてい、抽象構文木を評価 (eval) して生成します（場合によっては他の中間表現を挟んだりします）。

今回の処理系自作ではとりあえず、構文解析でいきなり Scheme オブジェクトを生成します。理由は、Scheme の抽象構文木は Scheme オブジェクト（S 式）とデータ構造が同じだからです。Scheme に限らず Lisp 系の言語は、この S 式の性質のおかげで、プログラムの実行時に構文木を動的に構築しやすいようになっています。この結果、マクロを実現するためにそこまで特別な仕組みを用意しなくても済みます。

Scheme でよく使われる構文の一部（`let` や `cond`, `begin` など）はマクロで記述できるので、処理系の実装から省くことができます。処理系自作の対象言語として Scheme を選択している理由の１つがこれです（実用上は処理系に組み込みで実装した方が性能が良いはずなので、手抜きです……！）。



## 字句解析器の実装

（[この段落に対応するコミット](https://github.com/PickledChair/fzscheme/commit/bffe580f4819d4622190de9cb1730d62c8ab352f)）

文字列そのままよりはトークン列に変換した方が構文解析の処理を書きやすいので、字句解析の処理を実装します。まずはトークンを表す構造体を定義します。

```c
typedef enum {
  TK_INT,
  TK_LPAREN,
  TK_RPAREN,
  TK_STR,
} TokenTag;

typedef struct Token Token;
struct Token {
  TokenTag tag;
  Token *next;
  char *loc;   // ソースコード中の元の文字列の先頭を指すポインタ
  int len;     // 元の文字列の長さ
  int64_t val; // 数値のトークンの場合、数値を代入する
  char *str;   // 文字列トークンの場合、文字列へのポインタを代入する
};
```

前回の Scheme オブジェクトの構造体と似た感じで、構造体の先頭にタグを付けて、トークンの種類を区別できるようにしています。また、トークン列を表現するのに連結リストを用いるので、 `next` メンバ変数で後続のトークンへアクセスできるようにしています。

ソースコードからトークン列を得るための処理を、とりあえず以下のように実装しました。大雑把でエラー処理もありませんが、とりあえず動くような実装です。

```c
static Token *new_token(TokenTag tag, char *start, char *end) {
  Token *tok = calloc(1, sizeof(Token));
  tok->tag = tag;
  tok->loc = start;
  tok->len = end - start;
  return tok;
}

Token *tokenize(char *input) {
  Token head = {};
  Token *cur = &head;

  while (*input) {
    // 空白文字をスキップ
    if (isspace(*input)) {
      input++;
      continue;
    }

    // 数値リテラル
    if (isdigit(*input)) {
      cur = cur->next = new_token(TK_INT, input, input);
      char *start = input;
      cur->val = strtoul(input, &input, 10);
      cur->len = input - start;
      continue;
    }

    // 文字列リテラル
    if (*input == '"') {
      char *start = input;
      while (*(++input) != '"');
      cur = cur->next = new_token(TK_STR, start, input);
      int str_len = input - (start+1);
      char *str_buf = calloc(1, str_len+1);
      strncpy(str_buf, start+1, str_len);
      str_buf[str_len] = '\0';
      cur->str = str_buf;
      input++;
      continue;
    }

    // 左括弧
    if (*input == '(') {
      cur = cur->next = new_token(TK_LPAREN, input, input+1);
      input++;
      continue;
    }

    // 右括弧
    if (*input == ')') {
      cur = cur->next = new_token(TK_RPAREN, input, input+1);
      input++;
      continue;
    }
  }

  return head.next;
}
```

（[chibicc](https://github.com/rui314/chibicc) のソースコードを読んだことのある方は、ほとんど劣化コピーじゃないかと思うかもしれません……。chibicc のソースコードは、コミット履歴と照らし合わせながら読むとよくわかるのですが、ものすごく綺麗なコードです。読むと勉強になると思うのでおすすめします。）

数値リテラルは今は整数のみの対応です。また、この実装だと負数を表現できないので、後で追加の実装をしたいですね（数値の前に `-` を付けた場合、多くの言語では `-` を単項演算子として処理すると思いますが、ここでは数値リテラルの一部として処理することになると思います）。

文字列リテラルの読み込みも今は手抜きで、ダブルクォートを見つけたら次のダブルクォートが見つかるまでを文字列リテラルの内容として直接採用しています。本当はエスケープ文字（`\t` や `\n` など）にも対応したいところですが、それは後回しにします。

これで字句解析がうまくいくかどうかをちょっと試してみます。`main` 関数に次の処理を追加してみました（別途 `print_token` 関数を実装しています）：

```c
int main(void) {
  char *source = "123 \"piyo\" \"hoge fuga\" (3 2 1)";
  Token *tokens = tokenize(source);
  for (Token *cur = tokens; cur != NULL; cur = cur->next)
    print_token(cur);
  free_token(tokens);

  ...

  return 0;
}
```

全然 Scheme の構文とは違う文字列ですが、字句解析はできます。以下のような結果になります：

```
INT     123
STRING  piyo
STRING  hoge fuga
LPAREN  (
INT     3
INT     2
INT     1
RPAREN  )
```


## 構文解析器の実装

（[この段落に対応するコミット](https://github.com/PickledChair/fzscheme/commit/b595402677713cab206f96ac6e9da3c8cd691e74)）

簡易的な字句解析器ができたところで、トークン列から Scheme オブジェクトを生成する構文解析器を実装してみます。

```c
Object *parse_objs(Token **tok) {
  Object *head = parse_obj(tok);
  if (head == NULL)
    return NIL;
  else
    head = new_cell_obj(head, NIL);

  Object *cur = head;
  for (;;) {
    Object *obj = parse_obj(tok);
    if (obj == NULL)
      return head;
    else
      cur = CDR(cur) = new_cell_obj(obj, NIL);
  }
}

Object *parse_obj(Token **tok) {
  if (*tok == NULL) return NULL;

  switch ((*tok)->tag) {
  case TK_INT: {
    Object *obj = new_integer_obj((*tok)->val);
    *tok = (*tok)->next;
    return obj;
  }
  case TK_LPAREN:
    *tok = (*tok)->next;
    return parse_objs(tok);
  case TK_RPAREN:
    *tok = (*tok)->next;
    return NULL;
  case TK_STR: {
    Object *obj = new_string_obj(strdup((*tok)->str));
    *tok = (*tok)->next;
    return obj;
  }
  }
}
```

構文解析器も例によって chibicc から実装のアイデアを得ています。具体的には `parse_objs` 関数と `parse_obj` 関数が `Token **` 型の引数を取るところです。`*tok = (*tok)->next`  でトークンの読み取り位置を一つ次にずらし、`*tok`  とアクセスすれば常に今から解析するトークン列の先頭を得られるようになっています。思いっきり副作用のある関数ですが、実装がシンプルになりコードが見やすくなるので真似してみました。

`parse_objs` 関数は前回少し説明した Scheme のコンスセル（`cell` のこと）を構築する関数です。コンスセルは実質連結リストのノードのことでした。実際、コンスセルの連なりのことを**リスト**と呼びます。また、連結リストの要素数がゼロの時、特別に**空リスト**と呼びます。空リストは `nil` とも呼ばれます。リストの最後の要素には必ず空リストがある決まりで、これがリストの終端を表します（でも仕組み上、連結リストの最後の要素が空リストじゃなくて、整数や文字列などの場合もありそうですよね。この場合はリストとは呼ばれず**ペア**という別の概念になります）。

このことから空のリストは沢山使われるので、空リストのためにその都度メモリ確保すると、メモリの無駄遣いになりそうです。いちいちメモリ確保しなくて良いように、`NIL` という名前で単一の空リストを定義して、空リストを使うときは必ずこれだけを使うようにしました。今回も、空のリストをパースしたときは `NIL` を返すようにしています。

構文解析器も今後拡張していくと思うのですが、実はおそらくこのコードより大幅に複雑になることはないと思います。『[Go言語でつくるインタプリタ](https://www.oreilly.co.jp/books/9784873118222/)』を読んだ方はわかると思いますが、同書籍の Monkey 言語のようなシンプルな言語であっても、手書きの再帰下降パーサがこんなにシンプルなコードになることは普通ありません。文が複数種類あったり、演算子の優先順位を考えたりするからです。

一方、Scheme の構文は S 式の１種類のみです。演算子も存在しません。演算子の役割は全て関数（Scheme では**手続き**と呼ばれる）に任せてしまいます。したがって構文解析の実装はとても容易で、処理系実装のこれ以降のステップに焦点を当てたい自分としては、この特徴がとてもありがたいです（つまりこれも Scheme を実装テーマに選んだ理由の１つです）。

構文解析がうまくいくかどうか試します。Scheme オブジェクトをプリントする関数は前回すでに実装したので、あとはループを回すだけです：

```c
int main(void) {
  char *srcs[] = {
    "42",
    "\"Hello, world!\"",
    "(1 2 3)",
    "()",
    "((1 2) (3 4) (5 6))",
  };

  for (int i = 0; i < sizeof(srcs) / sizeof(*srcs); i++) {
    Token *tok = tokenize(srcs[i]);

    for (Token *cur = tok; cur != NULL; cur = cur->next)
      print_token(cur);

    Token *tok_tmp = tok;
    Object *obj = parse_obj(&tok);
    if (obj != NULL) {
      print_obj(obj);
      putchar('\n');
      free_obj(obj);
    }
    tok = tok_tmp;

    free_token(tok);
  }

  return 0;
}
```

結果は以下のようになります：

```
INT     42
42
STRING  Hello, world!
Hello, world!
LPAREN  (
INT     1
INT     2
INT     3
RPAREN  )
(1 2 3)
LPAREN  (
RPAREN  )
()
LPAREN  (
LPAREN  (
INT     1
INT     2
RPAREN  )
LPAREN  (
INT     3
INT     4
RPAREN  )
LPAREN  (
INT     5
INT     6
RPAREN  )
RPAREN  )
((1 2) (3 4) (5 6))
```

最後の文字列は入れ子のリストにしてみました。これもうまく構文解析できていそうです。


## 次回の予定

前回書いたように **REPL (Read-Eval-Print Loop)** を実装しようと思っています。思いついたソースコードを解釈できるかどうか、その場ですぐに試せるようになって楽しそうです。

そのあとは通常の流れで行くと、もう少し字句解析を作り込んだり、識別子を読んでシンボルを生成できるようにしたり、構文解析結果を元に SECD マシンの命令列を出力するコンパイルの段階に進んだりすることになります。

しかし最近、「**まず簡易的な GC を作り始めると良いのでは？**」と思い始めました。処理系の作り込みから先に始めると、プログラムの全体的な複雑度が上がります。すると GC を実装する際になって、GC と処理系の仕組みをどう絡ませていくのか、考慮すべきことが最初から多くなりそうだと感じました。プログラムがシンプルな今のうちに、GC っぽいものを試作して色々失敗しておこうと思います。今回の最大の目的は GC の実装なので……！
