# ベアメタルでHello World
この章の内容は[The Embednomicon](https://docs.rust-embedded.org/embedonomicon/preface.html)にちょっと詳しい解説をつけただけです。すでにこちらのドキュメントを読んでいるのであればこの章は飛ばして次の章に進んでください。

## プロジェクトのセットアップ
`rustup`でRustコマンドをインストールすると`cargo`というコマンドが使えるようになります。
このコマンドはRust標準のビルドシステム兼パッケージマネージャーです。

まずはOS用の新規プロジェクトをつくりましょう。今回のプロジェクト名は`bookos`で行きます。もし自分の気に入ったプロジェクト名があれば、それでも構いません。
```
$ cargo new bookos
     Created binary (application) `bookos` package
$ cd bookos
$ ls
Cargo.toml  src
$ ls src
main.rs
```

`src/main.rs`をこれから書き換えてHello Worldするプログラムを書いていきます。
その前に、このプロジェクトでつかうRustのバージョンを指定するために`rust-toolchain`というファイルをこのプロジェクトにおいておきます。
```
$ echo "nightly-2019-09-19" > rust-toolchain
```

このファイルがプロジェクトのトップディレクトリにあることで、rustupにどのバージョンのRustを使えばいいかを自動的に教えて、バージョンを切り替えてくれるようになります。

## プログラムの実行
OSのある環境下では、たとえばターミナルから実行コマンドを打つことでOSがプログラムをロードして実行してくれますが、
OSのないベアメタル環境ではどのように自分の実行したいプログラムを実行すればいいのでしょうか？
電源投入時のCPUの動作について、ARMv7-MのアーキテクチャリファレンスマニュアルのB1.5.5　Reset behaviorに書いてあります。
この疑似コードによると、最後に`tmp`と`0xFFFF_FFFE`のandを取った値にブランチする、つまり`tmp`の値の最下位ビットを0にしたメモリアドレス上に置かれたプログラムが実行される、ということのようです。
この`tmp`は`vectortable`+4の値になるようですが、この`vectortable`の説明はB1.5.3にあります。Vector tableは例外が発生したときにどのアドレスにジャンプするべきかを指す配列です。
先頭エントリーはリセット時のスタックポインタの値になるので、4バイト先のエントリがリセット時に飛ぶべきアドレスとなっています。
このvector tableですが、どこに置かれるべきかはチップの実装依存となっているため、今回はSTM32F42xxxのリファレンスマニュアルを見る必要があります。
2.4 Boot configurationの章を見てみましょう。本来であれば0番地がコードの最初のエリアになるのですが、設定によってこれを変更できることがかてあります。
`BOOT0`と`BOOT1`の値で制御されるのですが、Nucleoボードのユーザーマニュアルによるとどうやらデフォルト値はともに0のようなので、メインのフラッシュメモリから読み出されることになります。
このフラッシュメモリのアドレスはというとSTMのリファレンスマニュアル3.3章の表に`0x800_0000`番地から始まる、とかいてあるので、ここにvector tableを配置してあげれば自分のプログラムを実行できるといことになりそうです。
このフラッシュメモリ領域がいわゆるROMの領域で、ここに自分のプログラムを書き込んであげることになります。

## no_std環境下でのRust
自分のプログラムをボード上で実行させる方法がなんとなくわかったと思いますが、問題はどうやったらこのフォーマットに従ったプログラムをRustで書いてバイナリにすればいいか、です。
それを解説したのがThe Embednomiconだったわけです。

通常、Rustのプログラムを書くときstdクーレートと呼ばれる言語の標準ライブラリを利用してプログラムをコンパイルすることになります。
しかし、このstdクレートは様々なOSの機能を利用することを前提として実装されているので、OSのないベアメタル環境下では当然動きません。
また、コンパイルしたときのフォーマットもOS上で実行するためのフォーマットになっているので、それも変更しなければなりません。
前者の問題を解決するための手段が`#![no_std]`アトリビュートです。これを使うとstdクレートの代わりに、OSの存在なしでも利用できる機能を提供する`core`クレートを使ってプログラムをコンパイルしてくれます。

では、no_stdでの最小プログラムをThe Embednomiconから引用します。
```
#![no_main]
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_panic: &PanicInfo<'_>) -> ! {
    loop {}
}
```

`#![no_main]`というアトリビュートがついていて、main関数がないこと、`#[panic_handler]`というアトリビュートがついた`panic`関数が存在することが普通のプログラムとは異なっています。
`main`関数が通常であれば最初に実行されるプログラムとなるのですが、これもOSが存在して事前準備をしてくれることを前提としたものになっています。
`main`関数に相当するレイヤーは後々つくっていくことになりますが、今回は最小の、なので省略です。
代わりにパニック時の動作についてはきちんと指定する必要があります。例えば`Option`の`None`に対して`unwrap`を呼び出すと通常のプログラムならば異常終了するはずですが、OSのないベアメタル環境ではこの異常終了時の動作を定義してあげる必要があります。
これが`#![panic_handler]`アトリビュートがついている`panic`関数、というわけです。今回は単に無限ループさせるだけですね。

このプログラムをビルドしてみましょう。
普通にビルドしてしまうと、実行しているPC向けのバイナリをつくってしまいますので、ターゲットを指定してあげる必要があります。
```
$ cargo build --target thumbv7em-none-eabihf
   Compiling bookos v0.1.0 (/home/garasubo/workspace/bookos)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
```
`thumb`というのはArmの命令セットの名前で、命令長が16ビットになっているという特徴があります。ARM命令と呼ばれる命令セットも存在してそちらは命令長が32ビットなのですが、ARMv7-Mではこちらはサポートされていません。
後ろの`hf`というのはハードウェアの浮動小数点演算器が存在することをしてしています。一部のARMv7-Mのチップでは浮動小数点演算器がないことがあり、コンパイラでそれらをソフトウェア演算にしなければならないことがあります。
もっとも、今回は浮動小数点の絡む処理を書く予定はないので、間違えてつけなくても（あるいは違うボードを使っていて浮動小数点演算器がないにもかかわらず`hf`をつけたとしても）問題にはならないでしょう。


## Vector tableを定義する
stdが使えない問題はクリアしましたが、まだなにもプログラムが実行できない状態なのでこれを解決していきましょう。
このArmマイコンではVector tableに従って最初に実行されるプログラムが決定されることがマニュアルからわかっているので、このVector tableを組み込んだプログラムをコンパイルできるようにすればこの問題は解決できます。
プログラムをコンパイルしたとき、各関数や変数がどのようなフォーマットでバイナリとして配置されるかを定義するにはリンカースクリプトというものを使います。
これはC言語など他の言語でベアメタルプログラミングするときも使うものです。
The Embednomiconの2.Memory layoutの章で使われているリンカースクリプトを見てみましょう。このリンカスクリプトは`link.ld`としてプロジェクトのトップディレクトリにおいておきましょう。
```
* Memory layout of the LM3S6965 microcontroller */
/* 1K = 1 KiBi = 1024 bytes */
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 64K
}

/* The entry point is the reset handler */
ENTRY(Reset);

EXTERN(RESET_VECTOR);

SECTIONS
{
  .vector_table ORIGIN(FLASH) :
  {
    /* First entry: initial Stack Pointer value */
    LONG(ORIGIN(RAM) + LENGTH(RAM));

    /* Second entry: reset vector */
    KEEP(*(.vector_table.reset_vector));
  } > FLASH

  .text :
  {
    *(.text .text.*);
  } > FLASH

  /DISCARD/ :
  {
    *(.ARM.exidx .ARM.exidx.*);
  }
}
```
`MEMORY`というのがマイコンのメモリのレイアウトを記述するセクションになっていて、`SECTIONS`でプログラムをそのメモリにどう配置するかを定義しています。
`MEMORY`の中身はハードウェア依存で、本書で使っているマイコンのレイアウトとは異なるので、あとで修正する必要があります。
`.vector_table`というのがVector tableを配置するところです。`FLASH`の先頭に置くようにしています。
一番最初のエントリーはスタックポインタのアドレスの初期値を格納する必要があります。
次のエントリーはリセット時に呼び出される関数へのアドレスになっています。これをRust側で定義してあげればいいというわけですね。
このスクリプトではRAM領域の末尾を指定してます。スタックポインタは末尾から先頭に向かって伸びていくので、通常は利用可能なRAM領域の末尾をしてしておけばよいでしょう。
`.text`セクションが実際のプログラムを置く場所です。
`.ARM.exidx`は標準ライブラリでのみ利用するセクションなので破棄するように指示しています。

では、`reset_vector`を定義してあげるところを引用します。
```
// The reset vector, a pointer into the reset handler
#[link_section = ".vector_table.reset_vector"]
#[no_mangle]
pub static RESET_VECTOR: unsafe extern "C" fn() -> ! = Reset;
```
`#[link_section = ".vector_table.reset_vector"]`というアトリビュートで、メモリ上の配置をしてあげることができます。
`RESET_VECTOR`という変数をそこにおいてあげるというわけですが、これが`Reset`関数への関数ポインタになっています。
`extern "C"`というのがついていると思いますが、これはC言語の関数と同じ形式で呼び出せるようにするというものです。
関数の型ですが、`() -> !`となっています。この返り値の`!`というのはこの関数を実行したら決して終了することはない、ということを意味しています。
C言語の関数とRustの関数はコンパイルしたときにフォーマットが異なり、C言語形式でないとマイコンはそこにジャンプしてそのまま実行ということができません。
`#[no_mangle]`というアトリビュートもついています。Rustでコンパイルしたとき、変数や関数の名前はリンカースクリプト内では別の名前（シンボル）に置き換えられるマングリングという処理が行われます。これを防ぐのがこのアトリビュートの役割です。
しかし、今回はこの変数を直接指定するということをしていないので、なくても動くでしょう。

あとは`Reset`関数を定義すれば完成です。
```
#[no_mangle]
pub unsafe extern "C" fn Reset() -> ! {
    let _x = 42;

    // can't return so we go into an infinite loop here
    loop {}
}
```

さて、上述のリンカスクリプトを使うためには、コンパイル時にオプションとして渡してあげる必要があります。
`RUSTFLAGS`を環境変数としてセットすることで、`rustc`コマンドにオプションを追加することができます。
```
$ RUSTFLAGS="-C link-args=-Tlink.ld" cargo build --target thumbv7em-none-eabihf
   Compiling bookos v0.1.0 (/home/garasubo/workspace/bookos)
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
```

さて、毎回このオプションを渡すのは少々面倒くさいです。`.cargo/config`というファイルをつくっておくと、デフォルトのパラメータをセットできます。詳しくは[公式のドキュメント](https://doc.rust-lang.org/cargo/reference/config.html)を参照してください。

```
[target.thumbv7em-none-eabihf]
rustflags = [
    "-C", "link-arg=-Tlink.ld",
]

[build]
target = "thumbv7em-none-eabihf"
```
こうしておけば単に`cargo build`とすれば前の様々なオプションをつけたコマンドと同じ結果が得られるはずです。


## プログラムをボードで実行する



## いざ、Hello World
さて、Hello Worldしましょう