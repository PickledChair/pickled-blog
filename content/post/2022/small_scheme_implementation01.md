---
title: "Scheme 処理系自作 (1) 動機と最初の実装"
url: "post/small_scheme_implementation01"
date: 2022-04-09T18:48:08+09:00
draft: false
description: "学習のために小さなScheme処理系を実装し始めました"
toc: true
categories:
  - "技術"
tags:
  - "Scheme"
  - "言語処理系自作"
share: true
---

## モチベーション

Scheme 処理系を C 言語で自作していこうと思います（実用のためではなく、学習のためのなんちゃって処理系、程度のものです）。

Scheme 処理系を自作するのはこれが初めてではなく、これまでに二度ほど実装（というより写経）してみたことがあります。しかし、その度に何かしらの心残りがありました：

- C で実装（参考実装：[kenpratt/rusty_scheme](https://github.com/kenpratt/rusty_scheme)）
	- C に慣れていなかった（というより今も慣れていない）ので、特にメモリ管理をどうすれば良いか方針を立てられなかった
		- 不慣れな C で実装したのは、当時読んでいた『[ゼロからのOS自作入門](http://zero.osdev.jp/)』の MikanOS 上で動作させたかったから。[実際に動作した](https://twitter.com/pickled_chair/status/1431531281653768192?s=20&t=gRfHdb9jyxAme5Zc-iP14g)
	- 雑に bdw-gc (Bohem GC) を導入してみたりしたが、それでもビジーループで大量のメモリを確保（数百MB単位）してしまう問題を解消できなかった（[当時のツイート](https://twitter.com/pickled_chair/status/1431541885193965569?s=20&t=4vuINLU2-QKPHqQwok7Ekw)）
		- 最初は素朴なツリーウォーク・インタープリタだったが、そのままでは末尾最適化対応が面倒だった
		- 末尾最適化を実現するために継続渡しスタイルで評価器を書き直したが、継続を表す構造体で余計に多くのメモリ確保を行ってしまった気がする
- Rust で実装（参考実装：[micro Scheme コンパイラの作成 - お気楽 Scheme プログラミング入門](http://www.nct9.ne.jp/m_hiroi/func/abcscm33.html)）
	- SECD マシン上で動作する Scheme コンパイラに挑戦。伝統的マクロも使える（[実装はここで公開しています](https://github.com/PickledChair/rusty_fzscheme)）
	- Rust の自動メモリ管理でメモリの問題も軽減されると期待したが、それでもそれなりに多くのメモリを使ってしまうようだった
		- オブジェクトを（仕方なく）clone する箇所が多かったので、そこで余計に多くメモリコピーが発生していたのかもしれない
		- そもそもオブジェクトをコピーすると、同じオブジェクトのはずなのに同一性が失われてしまうので、そこも問題があった（特にシンボルでこれが起こってほしくない）
		- Rust の言語的な制約で、不必要なオブジェクトのコピーをしないように実装するには、多分多くの unsafe な操作が必要になる気がする

だいたいメモリ管理に悩まされている感じです……。自分でメモリ管理していない分、自分でコントロールできなかったということでしょう。

ということで、メモリ管理についてもう少し真面目に調査したいというモチベーションが生まれました。今回の最大の目標は「**自分で GC を実装すること**」です。[以前も「GC を実装したい」と言っていた](https://twitter.com/pickled_chair/status/1431541885193965569?s=20&t=4vuINLU2-QKPHqQwok7Ekw)のですが、知識がなくてできていませんでした。今のところは素朴な Copying GC を考えています（これが一番簡単そうだったので）。

他にも、SECD マシン向けにはまだ末尾最適化に対応していなかったので、対応させたいと思っています。また、余裕があれば Unicode 対応もしたいですね。

プロジェクトの名称は **FZScheme** ということにしています。MikanOS で動かすことを当初の目的にしていたので、"**From Zero**" を冠しています。

## 最初の実装

『[低レイヤを知りたい人のためのCコンパイラ作成入門](https://www.sigbus.info/compilerbook)』や、先述の『OS自作入門』を見習って、最初はコンパイラとは言えないような小さな実装から始めていきたいと思います。[最初のコミットはこれです](https://github.com/PickledChair/fzscheme/commit/2cc1af4461edf36aa107ee80185c3c8e692617c6)。

Scheme のオブジェクトはだいたい以下のような感じで表現しようと思います。タグ付き構造体で、先頭のタグで型を判断し、それに応じて適切な共用体のメンバーを選ぶ感じです。

```c
typedef enum ObjectTag {
  OBJ_TAG_CELL,
  OBJ_TAG_INTEGER,
  OBJ_TAG_STRING,
} ObjectTag;

typedef struct Object Object;
struct Object {
  ObjectTag tag;
  union {
    struct {
      Object *car;
      Object *cdr;
    } cell;

    struct {
      long value;
    } integer;

    struct {
      char *value;
    } string;
  } fields_of;
};

#define CAR(obj) (obj)->fields_of.cell.car
#define CDR(obj) (obj)->fields_of.cell.cdr
```

Lisp をよく知らない場合「`cell` とは？　`cell` のメンバーにある `car` と `cdr` とは？」という疑問が湧くと思います。簡単に説明すると、`cell` は連結リストのノードで、`car` はノードの値、`cdr` は後ろ側のリストです。

（Scheme について詳しく知りたい場合、『[お気楽 Scheme プログラミング](http://www.nct9.ne.jp/m_hiroi/func/scheme.html)』というサイトがおすすめです。多分 Scheme 自体の知識に関しては今後もあまり説明していかないと思います。）

とりあえず、今回定義したオブジェクトで表現できるデータを印字できるようにしました。`main` 関数を以下のように書いて試しにオブジェクトを print してみます。

```c
Object *NIL = &(Object){OBJ_TAG_CELL};

int main(void) {
  Object *int_obj = new_integer_obj(42);
  print_obj(int_obj);
  putchar('\n');
  free_obj(int_obj);

  Object *str_obj = new_string_obj(strdup("Hello, world!"));
  print_obj(str_obj);
  putchar('\n');
  free_obj(str_obj);

  Object *list_obj = new_cell_obj(new_integer_obj(1), new_cell_obj(new_integer_obj(2), new_cell_obj(new_integer_obj(3), NIL)));
  print_obj(list_obj);
  putchar('\n');
  free_obj(list_obj);

  print_obj(NIL);
  putchar('\n');

  return 0;
}
```

実行すると、以下のような結果になります。

```scheme
42
Hello, world!
(1 2 3)
()
```

視覚的に結果が見えるのが好きなので、こんな感じで実装を始めてみました。差し当たっては次のように進めると思います。

- 入力文字列を字句解析・構文解析してオブジェクトを得られるようにする
- REPL (Read-Evaluate-Print-Loop) を実装して、標準入力で好きに入力した文字列を繰り返しオブジェクトに変換し、表示できるようにする

非常にまったり進めると思います。
