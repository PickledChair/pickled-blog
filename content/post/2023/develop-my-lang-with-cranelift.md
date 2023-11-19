---
title: "Cranelift による自作言語・コンパイラ入門をした"
url: "post/develop-my-lang-with-cranelift"
date: 2023-11-19T12:58:51+09:00
draft: false
description: "「Rustで作る！自作言語・コンパイラ入門」のコンパイラを Cranelift を使って実装した記録です"
toc: true
categories:
  - "技術"
tags:
  - "自作プログラミング言語"
  - "言語処理系自作"
  - "Rust"
  - "Cranelift"
share: true
---

## 「Rustで作る！自作言語・コンパイラ入門」を読んだ

先日、「Rustで作る！自作言語・コンパイラ入門」という本が X で話題になっていました。

{{< tweet user="cordx56" id=1723368127956676943 >}}

楽しそうだったので自分も読んでみました。本の内容としては、

- まずシンプルな言語仕様を決める
- その言語のソースコードを実行可能なインタプリタを実装する
	- JIT コンパイルでコード生成して即時実行する仕組み

というものでした。

言語仕様は本当にシンプルで、以下の３種類の文のみが存在します：

- 代入文（例： `x = 1` ）
- print 文（例： `print 3` ）
- if 文（例： `if x == 1 then print 2` ）
	- else 節はない
	- then 節には文が書けるが式は書けない

また、型や式に関しても制限が強いです：

- 型は符号なし 32 bit 整数と bool のみ
- 式は整数リテラルと識別子、そしてそれらを３種類の二項演算子 `+`, `-`, `==` と括弧 `()` で組み合わせたもの
	- bool のリテラル `true` や `false` などはない
	- つまり bool 値は `==` による整数どうしの等値比較によってのみ現れる
	- どの演算子もオペランドは整数の式のみをとる
	- つまり bool 値どうしを `==` 演算子で等値比較できない
- 代入文の右辺や print 文の引数に書けるのは整数の式のみ
	- つまり bool 値を変数に代入したり、print 文で表示したりできない
- if 文の条件式として書けるのは bool の式のみ

これらの制限によって、コンパイラの設計を格段にシンプルにできます。入門者向けの言語仕様として非常によく考えられていると感じました。

また、自分は自然言語でふんわり仕様を説明しましたが、本の中ではこの仕様の実現のために、型付け規則の定義や型推論の導入、EBNF による構文の定義など、より厳密な形式的説明を行っていました。初心者であってもこのような厳密な議論に慣れさせていくぞ、という本格派な趣向を感じました。

## Cranelift を使って実装してみた

この本の内容をもとに、実際にコンパイラを実装してみました。リポジトリは以下です。

https://github.com/PickledChair/simplelang

本では JIT コンパイルを [LLVM](https://llvm.org/)（の Rust 向け wrapper の [inkwell](https://github.com/TheDan64/inkwell) ）で実現していますが、自分の実装では [Cranelift](https://cranelift.dev/)[^cranelift] を使いました。Cranelift を使ったコンパイラの実装をしたことがないので練習台にしたい、というのがそもそもの実装の動機だったからです（LLVM が C++ で実装されているのに対して Cranelift は Rust で実装されているので、Rust から試しに使うなら Cranelift の方が導入がちょっと気軽です）。これだけシンプルな言語仕様であれば、コード生成部分の実装も簡単にできそうです。Cranelift の練習台にちょうど良いと思いました。

## 実装の解説

Cranelift は今のところ解説記事があまり存在しません。[cranelift-jit クレート](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/jit)の example や、[cranelift-jit-demo](https://github.com/bytecodealliance/cranelift-jit-demo) といったものを公式が用意してくれているので、JIT コンパイルの単純な例は学べます。しかし、もう少し発展的なことをしようとするとそれらの例だけでは足りないので、Cranelift を使用した実際の実装例を見つけて参考にする必要があります。自分の実装も多少は誰かの参考になればいいな……などと思っていたりします。そこで、自分が書いたソースから少し抜粋して解説しようと思います。

### 式のコード生成

実際のところ、Cranelift の使い方は多くの部分が LLVM と似通っているので、LLVM の場合と対応する API を見つけると上手く使えたりします。

次のコードは書籍に掲載されていた inkwell の使用例です。加算式の例が掲載されていました：

```rust
let i32_type = context.i32_type();
let result = match expr {
    Expression::Add(left, right) => {  
        let left = compile_expr(context, builder, left);
        let right = compile_expr(context, builder, right);
        builder.build_int_add(left, right)
    },
}
```

上のコードと対応する Cranelift 版の実装は以下のとおりです：

```rust
//  https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/codegen.rs#L167-L177
fn codegen_expr(&mut self, expr: &Expression) -> Value {
	match expr {
		/* 省略 */
		Expression::Add(lhs, rhs) => {
			let lhs = self.codegen_expr(lhs);
			let rhs = self.codegen_expr(rhs);
			self.func_builder.ins().iadd(lhs, rhs)
		}
		/* 省略 */
	}
}
```

「関数のビルダーがあって、加算命令を追加するメソッドを持っているのでそれを使う」というところが同じです（厳密にはちょっと異なっていますが）。このように、似ている部分を探して対応するように実装すると上手くいくこともあります。

### print 文のための関数

コード生成部分に関して、書籍ではコンパイラ実装に必要な例が直接的には書かれていません[^hard]（先ほどの加算式の場合だけが例外です）。その代わり、もう少し単純な例だけを示して、それを応用して実装するよう暗に促しています。

たとえば、書籍のコード例にあった LLVM における C 言語の printf 関数の使用例（以下）は print 文の実装のために使えます：

```rust
// printf関数を宣言  
let printf_fn_type = i32_type.fn_type(&[i8_ptr_type.into()], true);
let printf_function = module.add_function(
    "printf",
    printf_fn_type,
    None,
);

/* 省略 */

// printfをcall
builder.build_call(
    printf_function,
    &[hw_string_ptr.as_pointer_value().into()],
    "call",
);
```

一方、cranelift-jit でも libc の関数は呼び出せるものの、実はまだ C 言語の可変長引数の関数の呼び出しはサポートされていません（つまり printf 関数が呼べません）。cranelift-jit-demo の README.md に以下の説明があります：

> And to show off a handy feature of the jit backend, it can look up symbols with `libc::dlsym`, so you can call libc functions such as `puts` (being careful to NUL-terminate your strings!). Unfortunately, `printf` requires varargs, which Cranelift does not yet support.

ではどうするかというと、自分で固定長の引数の関数を定義して、それを JIT コンパイルした関数内から呼び出せるようにする方法を使います。

まず、u32 の数値を表示するだけの関数を定義します：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/jit_ctx.rs#L8-L10
fn println_u32(n: u32) {
    println!("{n}");
}
```

これにシンボルでアクセスできるように、`JITBuilder` でシンボル `println_u32` と関数ポインタを紐付けて、`JITModule` にその設定を渡します：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/jit_ctx.rs#L35-L40
let mut module = {
	let mut jit_builder = JITBuilder::with_isa(isa, default_libcall_names());
	let println_u32_addr: *const u8 = println_u32 as *const u8;
	jit_builder.symbol("println_u32", println_u32_addr);
	JITModule::new(jit_builder)
};
```

`JITModule` 側では `println_u32` の宣言だけを追加します。`Module::declare_function` の引数では、リンケージは `Linkage::Import` を設定します。これはちょうど、C 言語において関数のプロトタイプ宣言だけを書き、関数の実際のコードは他のオブジェクトファイルにあって、リンクすることでその関数が実際に使えるようになる、という流れと似ていると思います：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/jit_ctx.rs#L41-L45
let mut sig_println_u32 = module.make_signature();
sig_println_u32.params.push(AbiParam::new(types::I32));
let func_println_u32 = module
	.declare_function("println_u32", Linkage::Import, &sig_println_u32)
	.unwrap();
```

（注：関数のシグネチャで引数の型が `types::I32` になっていますが、これは i32 型であることを言いたいのではなく、単にサイズが 32 bit の数値であることを示しています。i32 と u32 の区別は Cranelift IR では存在しません）

上記のコード中の `func_println_u32` は `FuncId` です。print 文のコード生成では、call 命令に渡すのは `FuncRef` である必要がありますが、これは `Module::declare_func_in_func` で `FuncId` から変換して得ることができます：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/codegen.rs#L89-L95
fn codegen_print(&mut self, expr: &Expression) {
	let local_func = self
		.module
		.declare_func_in_func(self.print_func, &mut self.func_builder.func);
	let arg = self.codegen_expr(expr);
	self.func_builder.ins().call(local_func, &[arg]);
}
```

これで最初に定義した Rust 関数の `println_u32` を呼び出すことができます。

この方法は https://github.com/bytecodealliance/cranelift/issues/675 で説明されていた方法を参考にしました。

### 代入文の実装方法

今回の自分の実装は REPL で１つ１つ文を入力して、それぞれを逐次コンパイルする方法をとっています。もし REPL でなく、単一のソースを１度だけコンパイルして実行するのであれば、変数の実装は main 関数中のローカル変数とするので良さそうです。しかし自分は REPL を実装する方法を選んだので、一度定義した変数を後でコンパイルした関数から見えるようにするために、変数はグローバル変数としたいです。

ところで、ある変数を最初に定義して初期化する（初回の代入）のと、同じ変数にその後代入を行う（２回目以降の代入）で、処理が同じのはずがありません：

```
a = 1
print a
a = 2
print a
```

この例で、`a = 1` では 1 を格納するためのデータ領域を作りますが、`b = 2` ではすでにデータ領域があるので作りません。数値データの store はどちらの場合でも行います。

同じ変数に２度以上代入するのをエラーとする選択肢もありますが、今回は繰り返し代入するのを許容することにしました。`variables` というハッシュマップにこれまで定義した変数の情報を記録しておいて、すでに変数が存在しているかどうかを後で問い合わせ、処理を分岐させています：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/codegen.rs#L77-L84
Statement::Assign(ident, expr) => {
	let ident_str: &str = &*ident;
	if self.variables.contains_key(ident_str) {
		self.codegen_assign(ident, expr);
	} else {
		self.codegen_def_var(ident, expr);
	}
}
```

`codegen_def_var`（初回の代入のときの処理）は以下です。`DataDescription` はデータ領域を確保するためのビルダーのような役割を果たしています。`DataDescription::define` でバイト列を渡して初期化することもできるのですが、今回は `DataDescription::define_zeroinit` でゼロ初期化し、代入処理は `codegen_assign` の方に統一することにしました：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/codegen.rs#L114-L141
fn codegen_def_var(&mut self, ident: &Identifier, expr: &Expression) {
	let ident_str: &str = &*ident;
	let data = self
		.module
		.declare_data(ident_str, Linkage::Local, true, false)
		.unwrap();
	match expr {
		Expression::Comp(_, _) => unreachable!(),
		other => {
			self.data_description.define_zeroinit(4);
			self.module
				.define_data(data, self.data_description)
				.unwrap();
			self.variables.insert(ident_str.to_owned(), data);
			self.codegen_assign(ident, other);
		}
	}
	self.data_description.clear();
}
```

`codegen_assign`（初回と２回目以降の代入で共通の処理）は以下です：

```rust
// https://github.com/PickledChair/simplelang/blob/d121a9690fa023a28bab101aa4f7f0f1df92efe7/src/codegen.rs#L97-L112
fn codegen_assign(&mut self, ident: &Identifier, expr: &Expression) {
	let ident: &str = &*ident;
	let global_ref = {
		let data = *self.variables.get(ident).unwrap();
		let var = self
			.module
			.declare_data_in_func(data, &mut self.func_builder.func);
		self.func_builder
			.ins()
			.global_value(self.module.target_config().pointer_type(), var)
	};
	let value = self.codegen_expr(expr);
	self.func_builder
		.ins()
		.store(MemFlags::new(), value, global_ref, 0);
}
```

`variables` に記録してある `DataId` を取得して、`Module::declare_data_in_func` で `GlobalValue` を得ています。これをデータの保存先アドレスとして store 命令に渡すには `Value` に変換（load に相当）する必要があります。なぜこの処理が必要かというと、[Cranelift IR のドキュメント](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/docs/ir.md#global-values)では

> A _global value_ is an object whose value is not known at compile time. The value is computed at runtime by `global_value`, possibly using information provided by the linker via relocations. There are multiple kinds of global values using different methods for determining their value. Cranelift does not track the type of a global value, for they are just values stored in non-stack memory.

とあり、`GlobalValue` の値がコンパイル時にわからないからだ、とのことです。global_value 命令を使えば `GlobalValue` を `Value` に変換できます。

この `Value` はポインタ値であり、型は単に数値です。global_value 命令で指定する型は 64 bit マシンだけを考えると `types::I64` で決め打ちにしても良いのですが、`TargetFrontendConfig::pointer_type` を使えば、実行しているマシンのアーキテクチャに合わせてくれます。

## 自作言語・コンパイラ、しよう。

現在自分は趣味で他に自作言語のコンパイラ製作をしています（[今年初めの記事](/post/my_goals_for_2023)で考えていた通り）。ただ、自作言語はたいていどうしても実用性に欠けてしまいます。メジャーな言語のコンパイラや標準ライブラリなどの開発にたくさんの人の膨大な努力が注がれているのを見ると、やはり実用に足る言語はそう簡単には作れないことを思い知らされてしまいます。

しかし、コンパイラの自作を通して、既存の言語を眺めるときの解像度が上がることは間違いありません。ある言語のある仕様が気になったときに、「似たような機能を自分で実装したことがあったな……あれと結果が似ているな or あれとは結果が異なっているな」と思えれば、その類似性 or 差異を足がかりに言語仕様を深掘りして調べていくことができます。

（それと、ぶっちゃけると目的をこじつけたり、楽しいという理由だけで取り組んでも良いものだと思います。）

LLVM や Cranelift の IR はアセンブリ言語よりは抽象度が高く理解しやすいので、マシンコードを出力するようなコンパイラの実装の入門に良いのだと思います。一方、アセンブリ言語を出力するコンパイラに入門するには、「[低レイヤを知りたい人のためのCコンパイラ作成入門](https://www.sigbus.info/compilerbook)」に取り組むのも良いでしょう。

さて、今「自作コンパイラ」の話をしましたが、「自作言語」はまた別の要素があります。単にすでに仕様の決まっている言語のコンパイラを作るのではなく、言語仕様そのものから新たに決めていきます。この題材に関しては自分もまだまだ初心者で、最初の自作言語に取り組んでいる途中です。初めて真面目に構文や型システム、型推論を学んだり考えたりするので新鮮です。いつそれなりの形になるかわかりませんが、ちょっとずつ楽しんで作っていきたいと思います。

自作言語・コンパイラ、しよう。

[^cranelift]: LLVM と同様に、複数のアーキテクチャ向けのコード生成やコード最適化を引き受け、コンパイラ開発をフロントエンド部分の開発で済むようにできるライブラリ（コンパイラ基盤。 https://qiita.com/uint256_t/items/0a00497c689fc56fd5b4 に詳しい）。WebAssembly ランタイムの [wasmtime](https://wasmtime.dev/) や Firefox の JavaScript エンジン SpiderMonkey の裏側で wasm バイナリをマシンコードにコンパイルしています
[^hard]: そのため、実は本当の初心者にはちょっと難しい本なのではないかという気も少ししています。そこは読者を信頼しているということなのかもしれません
