---
title: "Ajisai コンパイラ育成日誌 第1日"
url: "post/growing-ajisai-001"
date: 2025-08-28T22:28:50+09:00
draft: false
description: "自作言語 Ajisai のコンパイラ育成日誌、第1日目の記録"
categories:
  - "技術"
tags:
  - "自作プログラミング言語"
  - "言語処理系自作"
  - "Ajisai"
share: true
---

ご無沙汰しておりました。気づけば 2025 年も半分以上が過ぎてしまいました。2025 年の目標記事は出さずじまいになってしまいましたが、2024 年の目標をそのまま引き継いでいる感じなので、結局のところ特に書くべきことはなかったと思います……。

さて、[2024年の目標](/post/my_goals_for_2024)に書いたように、[Ajisai](https://github.com/PickledChair/ajisai) という自作言語のコンパイラを少しずつ作っていました。しかし、さまざまな悩みや迷いが生じて、実装作業はなかなか進んでいませんでした。

迷いの最たるものは[実装言語を TypeScript から Swift に移行してみた](https://github.com/PickledChair/ajisai/pull/24)ことだったと思います。これには Ajisai の言語仕様（予定）と TypeScript の言語仕様が乖離していて、後で Ajisai 自体で記述し直すときの大変さを懸念したとか、TypeScript に不慣れだったのでより書きやすそうな言語を選び直したとか、それなりの目的はありました。そして結果的には、型推論の導入にある程度成功するという成果も挙げました。

しかし、最近また気が変わって、再び TypeScript で実装しようかと考え始めています。Swift は Swift で自分の使い方ではやはり辛さを感じる部分があったからです。特に、エラー伝播のために例外を使わずに Result 型だけを使うという縛りを課したら思いの外きつかった……。また、気軽に実行するのに Deno はやはり魅力的だというのもありました。Swift はコンパイル時間が長いですね……。

あと、Ajisai 言語の仕様でまた考え直したい部分があったというのもあります。別の言語で再実装していくと、コピペではなく、ある程度内容を理解しながら写さなければならないので、これまでの実装をもう一度把握しなおしたり、実装方法を再検討するのに良いのかな、となんとなく考えました。

ということで、TypeScript で実装し直すにあたって、**実装の記録もつけていこうかな**、という次第です。とりあえずまずやることは、C 言語のソースコードをファイルに出力して、それをコンパイルできるようにすることです（Ajisai 言語のターゲット言語はアセンブリ言語や LLVM IR ではなく C 言語なのです）。今回は以下のように出力ディレクトリ `ajisai-out` 以下に C 言語ソースおよびコンパイル結果の実行ファイルを出力するようにしました（`ajisai-out` ディレクトリ以下の構造は今後考えていきます）：

```ts
// C 言語のソースコード
const cSource = `#include <stdio.h>

int main() {
  printf("%d\\n", 42);
  return 0;
}
`;

// 出力ディレクトリ ajisai-out を準備
const outputDirName = "ajisai-out";
try {
  const distDirStat = Deno.statSync(outputDirName);
  if (!distDirStat.isDirectory) {
    console.error(`"${outputDirName}" found, but not a directory`);
    Deno.exit(1);
  }
} catch {
  Deno.mkdirSync(outputDirName);
}

// ajisai-out/main.c に C ソースコードを書き込み
const outputCFilePath = `${outputDirName}/main.c`;
try {
  const outputCFile = Deno.openSync(
    outputCFilePath,
    { write: true, create: true, truncate: true },
  );
  const encoder = new TextEncoder();
  outputCFile.writeSync(encoder.encode(cSource));
} catch {
  console.error(`couldn't write C source to "${outputCFilePath}"`);
  Deno.exit(1);
}

// C ソースをコンパイルして実行ファイル ajisai-out/main を出力
const outputBinFilePath = `${outputDirName}/main`;
const command = new Deno.Command("cc", {
  args: ["-o", outputBinFilePath, outputCFilePath],
  stdout: "inherit",
  stderr: "inherit",
});
const { code } = command.outputSync();
Deno.exit(code);
```

ただ「42」とプリントするだけの実行ファイルをコンパイルするプログラムです。こうやってスクリプト言語よろしくベタ書きで気軽に書いていけるのは楽ですね。実装の記録は今後 [PickledChair/growing-ajisai](https://github.com/PickledChair/growing-ajisai) リポジトリに置いていきたいと思います（今回のコードは [day001-010/day001](https://github.com/PickledChair/growing-ajisai/tree/main/day001-010/day001) ディレクトリにあります）。最終目標は Ajisai 言語自体による Ajisai 言語コンパイラの記述（**セルフホスト**）です。進捗が出たり出なかったりだと思いますが、今度は最後まで続くといいな。
