---
title: "Ajisai コンパイラ育成日誌 第2日"
url: "post/growing-ajisai-002"
date: 2025-08-30T20:37:43+09:00
draft: false
description: "自作言語 Ajisai のコンパイラ育成日誌、第2日目の記録"
categories:
  - "技術"
tags:
  - "自作プログラミング言語"
  - "言語処理系自作"
  - "Ajisai"
share: true
---

[前回](/post/growing-ajisai-001)は、以下のC言語のソースコードをファイルに書き出して、それをコンパイルするだけのプログラムを書きました：

```ts
// C 言語のソースコード
const cSource = `#include <stdio.h>

int main() {
  printf("%d\\n", 42);
  return 0;
}
`;
```

まだ Ajisai 言語がどんな言語仕様なのかといった議論は何もしておらず、無から定型の実行ファイルが生成されるだけの、全く自由のないコンパイラ（？）です。ここで、とりあえず整数値の `42` の部分だけを抜き出して、Ajisai 言語のソースコードと言い張ってみましょう：

```ts
// Ajisai 言語のソースコード
const ajisaiSource = "42";

// C 言語のソースコード
const cSource = `#include <stdio.h>

int main() {
  printf("%d\\n", ${ajisaiSource});
  return 0;
}
`;
```

ちょっと進歩が見られましたが、コンパイラのユーザーがわざわざコンパイラのソースコードを開いて `ajisaiSource` の値を変更しないと、好きな整数値に変更できませんね。`ajisaiSource` に与える文字列を、外部のファイルから取得できるように変更してみます：

```ts
// Ajisai 言語のソースコードをファイルから読み込む
const ajisaiSource = Deno.readTextFileSync(sourceFilePath);
```

しかし、`sourceFilePath` はどうやって取得しましょう？　`Deno.args` から直接コマンドライン引数を取得しても良いのですが、引数の解析が面倒そうです……。良さげな argument parser ライブラリの [cliffy](https://cliffy.io/) を見つけたので使ってみることにしました：

```ts
import { Command } from "jsr:@cliffy/command@^1.0.0-rc.8";

// build コマンドのオプションはなし
type Options = Record<string, never>;
// build コマンドの引数: <sourceFilePath:string>
type Arguments = [string];

// サブコマンド build の処理
function buildMain(_options: Options, ...args: Arguments) {
  const [sourceFilePath] = args;
  // ...
}

const build = new Command()
  .arguments("<source:string>")
  .description("Create executable from source files.")
  .action(buildMain);

await new Command()
  .name("ajisai")
  .version("0.0.1")
  .description("Ajisai compiler")
  .command("build", build)
  .parse(Deno.args);
```

そして、これまで書いてきた処理は `buildMain` 関数の中に移します。それから、その処理の前段階で `sourceFilePath` の指すファイルからテキストを読んで `ajisaiSource` 変数の値として設定します：

```ts
function buildMain(_options: Options, ...args: Arguments) {
  // sourceFilePath の存在確認
  const [sourceFilePath] = args;
  try {
    const sourceFileStat = Deno.statSync(sourceFilePath);
    if (!sourceFileStat.isFile) {
      console.error(`"${sourceFilePath}" found, but not a file`);
      Deno.exit(1);
    }
  } catch {
    console.error(`"${sourceFilePath}" not found`);
    Deno.exit(1);
  }
  // Ajisai 言語のソースコードをファイルから読み込む
  const ajisaiSource = Deno.readTextFileSync(sourceFilePath);

  // ...
}
```

使ってみましょう。試しにソースファイルを `examples/answer.ajs` という名前で作って、そこからコンパイルしてみます：

```
$ deno -A ajisai.ts --help

Usage:   ajisai
Version: 0.0.1

Description:

  Ajisai compiler

Options:

  -h, --help     - Show this help.
  -V, --version  - Show the version number for this program.

Commands:

  build  <source>  - Create executable from source files.

$ cat examples/answer.ajs
42
$ deno -A ajisai.ts build examples/answer.ajs
$ ./ajisai-out/main
42
```

うまくいってそうですね。

ところで、現状は Ajisai のソースコードをそのままC言語のソースコードの中に埋め込んでいるだけです。ということは、Ajisai の整数演算の仕様がC言語と同じだと仮定すると、ソースコードとして算術演算を書いてみてもうまくいきそうです。やってみましょう：

```
$ cat examples/arith.ajs
(10 + 11) * 2
$ deno -A ajisai.ts build examples/arith.ajs
$ ./ajisai-out/main
42
```

うまくいきました。しかも、構文エラーもCコンパイラが報告してくれます（当たり前ですが……）：

```
$ cat examples/invalid.ajs
(10 + 11) *
$ deno -A ajisai.ts build examples/invalid.ajs
ajisai-out/main.c:5:1: error: expected expression
    5 | );
      | ^
1 error generated.
```

今はこれでうまくいっていますが、今後は間違いなく言語仕様がC言語とは異なってくるので、構文解析や意味解析を自前でやっていく必要がありますね。次回からはそれに取り組んでいきたいと思います。

（今回のコード: [day001-010/day002](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day002) ）

