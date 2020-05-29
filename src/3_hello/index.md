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
その前に、このプロジェクトで使うRustのバージョンを指定するために`rust-toolchain`というファイルをこのプロジェクトにおいておきます。
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
2.4 Boot configurationの章を見てみましょう。本来であれば0番地がコードの最初のエリアになるのですが、設定によってこれを変更できることが書いてあります。
`BOOT0`と`BOOT1`の値で制御されるのですが、Nucleoボードのユーザーマニュアルによるとどうやらデフォルト値はともに0のようなので、メインのフラッシュメモリから読み出されることになります。
このフラッシュメモリのアドレスはというとSTMのリファレンスマニュアル3.4章の表に`0x800_0000`番地から始まる、と書いてあるので、ここにvector tableを配置してあげれば自分のプログラムを実行できるということになりそうです。
このフラッシュメモリ領域がいわゆるROMの領域で、ここに自分のプログラムを書き込んであげることになります。

## no_std環境下でのRust
自分のプログラムをボード上で実行させる方法がなんとなくわかったと思いますが、問題はどうやったらこのフォーマットに従ったプログラムをRustで書いてバイナリにすればいいか、です。
それを解説したのがThe Embednomiconだったわけです。

通常、Rustのプログラムを書くときstdクレートと呼ばれる言語の標準ライブラリを利用してプログラムをコンパイルすることになります。
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
後ろの`hf`というのはハードウェアの浮動小数点演算器が存在することを示しています。一部のARMv7-Mのチップでは浮動小数点演算器がないことがあり、コンパイラでそれらをソフトウェア演算にしなければならないことがあります。
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
関数の型ですが、`() -> !`となっています。この返り値の`!`というのはこの関数を実行したら決して終了することはない、ということを意味しています（発散する関数とも言います）。
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
ここで一度ボードにプログラムを書き込んであげたいと思います。
まずは、`link.ld`の中のアドレスを修正するところから始めます。STMのリファレンスマニュアルの3.4章によるとFLASHの領域は`0x800_0000`から`0x81f_ffff`までの2MB領域に広がっていることがわかります。
RAM領域はリファレンスマニュアルの2.3.1を見ると`0x2000_0000`から256KBの領域にあるとわかります。
しかし、STMのデータシートの方を3.6章を見ると、64KBはcore coupled memoryで使うことができず、5章のメモリマップを見ると`0x2003_0000`以降の領域は使えないことがわかります。
よって、以下のように書き換えましょう。
```
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 2M
  RAM : ORIGIN = 0x20000000, LENGTH = 192K
}
```
書き換えたら再度`cargo build`しましょう。

プログラムをフラッシュROMに書き込むにはST-Linkというこのボードに内蔵されたデバッグ用インターフェースを利用することで可能です。
ボードには2つmicro USBの端子がついていると思いますが、切れ込みで別れた小さいエリアについている方の端子（CN1）がST-Linkと通信するための端子です。
PC側はこのインターフェースを利用するために[Open OCD](http://openocd.org/)を使います。Ubuntuであれば`apt`コマンド経由でインストールできます。
```
IMAGE=target/thumbv7em-none-eabihf/debug/bookos openocd -f interface/stlink-v2-1.cfg -f target/stm32f4x.cfg -c "init; reset halt; flash write_image erase $IMAGE; verify_image $IMAGE; reset; shutdown"
```
`-f`オプションでデフォルトで用意された設定ファイルを利用することができます。`-c`オプションでコマンドを実行しています。このコマンドでボードを初期化した後、`$IMAGE`で指定されたファイルを書き込んだ後、ちゃんと書き込めたかの確認もしています。
`cargo build`で生成されたバイナリは`target/<アーキテクチャ名>/debug/<アプリケーション名>`にあります。
しかし、もとのプログラムが無限ループするだけで何もしていないプログラムのため、これでは動いているかどうかすらよくわかりません。
そこで、デバッガを利用することでどのようにプログラムが動いているか覗いてみましょう。デバッガは`GDB`を利用します。
Ubuntu 18.04以降では`apt`より`gdb-multiarch`を、それ以前の場合は`gdb-arm-none-eabi`をインストールしましょう。
GDBには別の環境で動いているプログラムと通信してデバッグする機能があります。Open OCDはGDBと通信するためのサーバーとしての機能もあります。
ターミナルを2つ開いて、片方のターミナルでは
```
openocd -f interface/stlink-v2-1.cfg -f target/stm32f4x.cfg
```
を実行しておくとこれがGDBのサーバーとなります。
別のターミナルで
```
gdb-multiarch target/thumbv7em-none-eabihf/debug/bookos
```
を実行するとGDBが立ち上がりますが、この状態ではまだサーバーとは通信していません。`target remote`コマンドでサーバーを指定したのち、プログラムを読み込んでみましょう。
```
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
(gdb) load
Loading section .vector_table, size 0x4 lma 0x8000000
Start address 0x0, load size 4
Transfer rate: 7 bytes/sec, 4 bytes/write.
(gdb) break Reset
Breakpoint 1 at 0x800000c: file src/main.rs, line 13.
```
これでReset関数にブレークポイントが仕掛けられました。
あとは普通にステップ実行できます。
```
(gdb) continue
Continuing.
Note: automatically using hardware breakpoints for read-only addresses.

Breakpoint 1, Reset () at src/main.rs:13
13	    let _x = 42;
(gdb) step
16	    loop {}
(gdb) 
^C
Program received signal SIGINT, Interrupt.
0xe7ff9000 in ?? ()
```
最後は無限ループになっているので、強制的に止めました。


## いざ、Hello World
プログラムは書き込めたのでHello Worldを出力していきたいと思います。
しかし、今回のボードにはディスプレイはついていません。どこにこの文字を出力すればいいのでしょうか。
一般的に、このような組込みボードと通信するインターフェースとしてUARTというモジュールがあります。これを使えばPCとマイコン間で比較的簡単に通信ができます。
しかし、今回はこれを使わずもっと手軽なセミホスティングというデバッガを介した出力でこれをやろうと思います。
この方法はあくまでデバッグ用の機能なので、デバッガをつけられない実際の製品版で用いることはできず、UARTと比べるとかなり通信速度は遅いのですが、
今回の目的はあくまでプログラムがちゃんと動作しているかの確認のためのHello Worldなので、セミホスティングを使うことにしましょう。
このセミホスティングの機能を全部理解するのも大変なので、今回は既存のライブラリを使ってしまいましょう。
RustのEmbeddedワーキンググループの提供しているクレートである`cortex-m-semihosting`を使います。
`Cargo.toml`の`dependency`としてこのクレートを追加します。
```
[dependencies]
cortex-m-semihosting = "0.3.5"
```
その後、`src/main.rs`内のReset関数を以下のように書き換えます。

```
use cortex_m_semihosting::hprintln;

#[no_mangle]
pub unsafe extern "C" fn Reset() -> ! {
    hprintln!("Hello World").unwrap();

    loop {}
}
```
これでビルドをしてみると、リンクのところで失敗してしまいます。
```
$ cargo build
...
  = note: rust-lld: error: no memory region specified for section '.rodata'
          rust-lld: error: no memory region specified for section '.bss'
          

error: aborting due to previous error

error: Could not compile `bookos`.

To learn more, run the command again with --verbose.
```
`.rodata`と`.bss`セクションがないと言われています。ざっくり説明すると`.rodata`は定数を保存しておく領域、`.bss`は初期値が0のグローバル変数を保存する領域です。
これ以外にも`.data`という初期値が0でないグローバル変数のための領域も実は定義をサボってきました。
これらのセクションはリンカースクリプトで定義しないといけないのはもちろんなのですが、これをちゃんとプログラムで使うためにはいくつか処理をする必要があります。
`.bss`セクションは書き換えの必要があるため、RAM領域に配置されるのですが、RAM領域の初期値が0である保証はないのでこれを0にプログラム側で初期化してあげる必要があります。
`.data`セクションは初期値がROM領域にあって実際の変数はRAM領域に配置されることになるので、ROM領域の初期値をRAM領域側にコピーする必要があります。
`.rodata`はリードオンリーなので何もしなくて大丈夫です。
Embedonomiconと同様にReset関数の先頭で以下のような処理をしておきましょう。

```
use core::ptr;

pub unsafe extern "C" fn Reset() -> ! {
    extern "C" {
        static mut _sbss: u8;
        static mut _ebss: u8;
        static mut _sidata: u8;
        static mut _sdata: u8;
        static mut _edata: u8;
    }

    let count = &_ebss as *const u8 as usize - &_sbss as *const u8 as usize;
    ptr::write_bytes(&mut _sbss as *mut u8, 0, count);

    let count = &_edata as *const u8 as usize - &_sdata as *const u8 as usize;
    ptr::copy_nonoverlapping(&_sidata as *const u8, &mut _sdata as *mut u8, count);
...
```

`link.ld`は以下のようなセクションを加えます。
```
SECTIONS
{
...

  .rodata :
  {
      *(.rodata .rodata.*);
  } > FLASH

  .bss (NOLOAD):
  {
    _sbss = .;
    *(.bss .bss.*);
    _ebss = .;
  } > RAM

  .data : AT(ADDR(.rodata) + SIZEOF(.rodata))
  {
    _sdata = .;
    *(.data .data.*);
    _edata = .;
  } > RAM

  _sidata = LOADADDR(.data);

  /DISCARD/ :
...

```

link.ldで定義した定数をReset関数で`extern C`で読み込んで使っています。
まず`_sbss`から`_ebss`の領域を`ptr::write_bytes`で0で初期化します。
次に`.data`の初期値は`_sidata`の値から始まるROM領域に存在しますがROM領域にはプログラムでは書き換え不能なので、プログラム上の配置は`_sdata`から始まるRAM領域になっています。
これを`ptr::copy_nonoverlapping`でコピーします。

さて、これで今度はcargo buildでのコンパイルが通るはずです。
さっきと同様に、Open OCDとGDBを立ち上げてプログラムを実行すると、Open OCD側にHello Worldが出現するのが期待される結果ですが、そのためにはOpen OCDのセミホスティング機能を有効化する必要があります。
GDBを立ち上げ`target remote :3333`を実行したあとに、
```
(gdb) monitor arm semihosting enable
```
というコマンドを実行してから、`continue`を実行してプログラムを再開すると、以下のようにOpen OCD側にHello Worldが出力されるはずです。
```
...
xPSR: 0x01000000 pc: 0x08000008 msp: 0x2001c000
in procedure 'arm'
semihosting is enabled
Hello World

```
