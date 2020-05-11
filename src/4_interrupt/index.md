# 割り込み制御
割り込みはOSをつくる上で重要な項目の1つです。
ペリフェラルが割り込み信号を送ることで、OSは現在の実行処理を中断してペリフェラルからの処理をすることができます。

Armでは現在の実行を中断して非同期に発生するイベントを「例外処理」と呼んでいて、割り込みもその１つです。
今回はSysTickというArmに内蔵されているタイマーモジュールを例として扱ってみようと思います。
SysTickはシステムタイマーといわれていて、OSで一定周期ごとに行う処理などを実装するモジュールです。
このモジュールはARMv7-Mでは必須のモジュールであるため、今回使うNucleoのボード以外のARMv7-Mのボードでも存在するはずです。

## ARMv7-Mの例外モデル
ARMv7-MのリファレンスマニュアルB1.5章に例外処理の詳細が書かれています。
例外が発生すると、前の章で説明したVector Tableに従って処理が決定されます。
割り込みを含む例外が発生すると、例外に割り振られたIDによってVector Tableに書かれたアドレスにジャンプします。
今回扱うSysTickモジュールではB1.5.2章によると15番の例外が発生します。つまり、Vector Tableの15番目のエントリーとして呼びたい関数をおいておくことになります。

例外が発生した際、後にもとのプログラムの実行に戻るため、プロセッサの状態を一部保存する必要があります。具体的にはArmのレジスタをメモリに退避させることになります。
ここでArmのレジスタについて簡単に説明します。詳しくはB1.4章に書かれています。
ARMv7-MではR0からR15までのレジスタが使用可能です。R0からR12レジスタが汎用レジスタと呼ばれるもので、プログラム中で自由に使うことができます。
R13からR15も算術命令などで使うこともできるのですが、それぞれ特殊な意味を持つレジスタになっています。
R13がスタックポインタ（SP）になっています。現在のスタック領域のメモリアドレスを指すものです。モードによって実は２つのスタックポインタレジスタが使い分けられています。
つまり、プロセッサのモードによって一見アセンブラ命令上は同じSPでも異なるSPを指すことがあるということです。詳しくは次の章で説明します。
R14はリンクレジスタ（LR）です。これは関数の呼び出しをおこなった場合に使われるもので、その関数呼び出しから戻る先のアドレスを保存するためのレジスタです。
R15はプログラムカウンタ（PC)で、現在、実行中の命令のメモリアドレスが格納されているレジスタです。
これ以外のレジスタとしていくつかシステムレジスタが存在しています。これは通常の算術命令などではアクセスできず、特別な命令でしかアクセスできません。
そのうちの１つがプログラムスタータスレジスタ（PSR）というものです。このレジスタは32ビットのレジスタなのですが、用途に合わせて3つのレジスタに分解されるというものです。詳細はここでは割愛します。

例外が発生するとき、ハードウェアがシステムの状態を自動的にスタック領域に退避してくれます。B1.5.6章を見てみましょう。
スタックにはPSR、例外処理が完了したあとに戻るべきアドレス、LR（R14）、R12、R3からR0までが退避されます。また、浮動小数点拡張が有効になっている場合、さらに浮動小数点に関するレジスタも退避されますが、今回は有効にしていないので気にしなくても大丈夫です。
例外処理から復帰する際、これらのスタックに保存されたものが自動的にレジスタに書き戻され、スタックポインタの位置も戻される、という処理もなされます。
しかし、スタック領域に保存されない汎用レジスタが存在することに注意が必要です。これらは通常のプログラムで使用される可能性が十分にあります。
スタックポインタは通常の関数呼び出しをして戻ってくる際に元の値に戻されるはずなので問題ないでしょう。
プログラムカウンタも完了したあと戻るべきアドレスがスタック上に保存されていて、それがプログラムカウンタに書き戻されるのでこれも大丈夫です。
残りのR4からR11のレジスタが使われる可能性があります。しかし、これらのレジスタはArmの関数の呼び出し規約により、関数を呼び出しても関数側で値を戻す必要があるのでこれも保存しなくても大丈夫です。
よって、割り込みハンドラとして関数を呼び出して戻るだけなら、ハードウェア側で必要なレジスタはすべてスタックに保存される、ということになります。

割り込みハンドラ関数から通常の処理に復帰する際にはPCに通常のアドレスではなく、特殊な値を書き戻すことによって通常モードに戻ることができます。
これがB1.5.8で説明されていることです。しかし、B1.5.6章を見ると、割り込みが発生する際、LRの値がもとのモードに戻るように設定されることがわかります。
つまり、特に何もしなくても通常の処理に戻れそうです。

## SysTickの割り込みハンドラを定義する
では、SysTickの割り込みハンドラを定義しましょう。今回は割り込みハンドラの中で特に特別なことはせず、適当な文字列をセミホスティングで表示させるだけにしましょう。
Embedonomiconの4章にあるコードを`main.rs`に貼り付けてそのまま使ってしまいましょう。
```
pub union Vector {
    reserved: u32,
    handler: unsafe extern "C" fn(),
}

extern "C" {
    fn NMI();
    fn HardFault();
    fn MemManage();
    fn BusFault();
    fn UsageFault();
    fn SVCall();
    fn PendSV();
    fn SysTick();
}

#[link_section = ".vector_table.exceptions"]
#[no_mangle]
pub static EXCEPTIONS: [Vector; 14] = [
    Vector { handler: NMI },
    Vector { handler: HardFault },
    Vector { handler: MemManage },
    Vector { handler: BusFault },
    Vector {
        handler: UsageFault,
    },
    Vector { reserved: 0 },
    Vector { reserved: 0 },
    Vector { reserved: 0 },
    Vector { reserved: 0 },
    Vector { handler: SVCall },
    Vector { reserved: 0 },
    Vector { reserved: 0 },
    Vector { handler: PendSV },
    Vector { handler: SysTick },
];

#[no_mangle]
pub extern "C" fn DefaultExceptionHandler() {
    loop {}
}
```

`extern C`によってSysTick関数やそれ以外の例外ハンドル用の関数を宣言しています。
これはC言語などで書かれた関数を呼び出すものです。よって、リンクする際にどこかから引っ張ってこなければなりません。
そのため、Embedonomiconでは以下のようなセクションをリンカスクリプトに加えています。

```
PROVIDE(NMI = DefaultExceptionHandler);
PROVIDE(HardFault = DefaultExceptionHandler);
PROVIDE(MemManage = DefaultExceptionHandler);
PROVIDE(BusFault = DefaultExceptionHandler);
PROVIDE(UsageFault = DefaultExceptionHandler);
PROVIDE(SVCall = DefaultExceptionHandler);
PROVIDE(PendSV = DefaultExceptionHandler);
PROVIDE(SysTick = DefaultExceptionHandler);
```
これは、該当する関数が見つからなかった場合、`DefaultExceptionHandler`を代わりに使うという文になっています。
さらに、`EXCEPTIONS`を`RESET_VECTOR`直後に置くため、`.vector_table`セクションも以下のように書き換える必要があります。

```
  .vector_table ORIGIN(FLASH) :
  {
    LONG(ORIGIN(RAM) + LENGTH(RAM));

    KEEP(*(.vector_table.reset_vector));

    KEEP(*(.vector_table.exceptions));
  } > FLASH
```

これで、とりあえず、ビルドは通りますが、実行しても`SysTick`関数を定義していないので、ただの無限ループである`DefaultExceptionHandler`を使うだけなのでなにもわかりません。
そのうえ、まだSysTick割り込みを発生させるためのSysTickモジュールの設定もしていないので、これも行う必要があります。

それではまず、SysTick関数を定義しておきましょう。この関数はあとでリンカスクリプトから参照され、割り込みが発生したときC言語の関数呼び出しのように呼ばれるので、
`DefaultExceptionHandler`と同様に`no_mangle`と`extern "C"`をつけておく必要がある点に注意しましょう。
```
#[no_mangle]
pub extern "C" fn SysTick() {
    hprintln!("Systick").unwrap();
}
```

この関数を定義したあと、先程加えた`extern "C"`で各種例外ハンドル用関数を宣言していたところから`SysTick`の部分を取り除く必要もあります。

## SysTickモジュールを定義する
SysTickの例外ハンドラが定義できたところで、実際のSysTick例外を発生させるところにチャレンジしましょう。
ところで、今まですべての関数をすべて`main.rs`に書いてきてしまいました。Embedonmiconではいくつかのサブプロジェクトにわけてクレートとして分離する、という方法をとってコードを整理しています。
同じようにクレートとして分離するのも悪くないのですが、今回はお手軽にモジュールという形で分離しましょう。
詳しくは[TPRLの7章](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)が参考になると思います。
このモジュールのシステムは2018 Editionで大きく変更がされているため、個人のブログなどで2015 Editionをベースに解説してる場合があることに注意が必要です。
ただし、後方互換性はあるので、2015 Editionに沿った方法でモジュールを分割しても大きな問題にはなりません。

`src`ディレクトリ以下に`systick.rs`という以下のような関数を含んだファイルをつくりましょう。
```
use cortex_m_semihosting::hprintln;

pub fn init() {
    hprintln!("Systick init").unwrap();
}
```

続いて、`main.rs`内の`Reset`関数を以下のように書き換えます。
`systick`モジュールの宣言と、`systick`モジュール内の`init`関数を呼び出しています。
```
mod systick;

#[no_mangle]
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

    hprintln!("Hello World").unwrap();

    systick::init();

    loop {}
}
```

ここまでで一度ビルドして実行すると新たに`Systick init`という文字列が表示されるはずです。
ここからSysTickに関係するコードはこの`systick.rs`内に書いていき、`systick`モジュールとして`main.rs`から使いましょう。

## SysTickのレジスタを設定する
SysTickを制御するにはSysTickのレジスタを叩いてあげる必要があります。
ArmではこのようなモジュールやペリフェラルのレジスタをメモリマップドIOという形で読み書きします。
プログラム側からは普通のメモリアクセスと同じようにできるという点で便利な仕組みです。
メモリマップドIOとは、このような外部のデバイスのレジスタをメモリアドレスに割り当てて、メモリアクセスのようにレジスタを読み書きする仕組みです。
ただし、これらのレジスタの値の読み書きは本物のメモリとは違って例えば、一部のビットが書き込み不可であるアドレスに書き込んだあとすぐに読み込んだら値が一致しないとか、何も書き込んでいないのに時間が立つと値が変わっている、などの現象が発生します。
そのため、コンパイラが普通のメモリアクセスと同じような最適化をかけてしまうと正しく動作しない場合があります。ここには注意しましょう。

SysTickの詳しい説明はリファレンスマニュアルのB3.3章にまとまっています。
SysTickを使うにはコントロールレジスタ（CSR）、リロードバリューレジスタ（RVR）、カレントバリューレジスタ（CVR）、キャリブレーションバリューレジスタ（CALIB)の3つを触ることになります。
CSRはこのSysTickを有効化させたり、割り込み発生をさせるかどうかの制御などをするためのものです。
CVRは現在のタイマーの値で時間経過とともに値が減っていきます。この値が0になると割り込みを発生させることができます。
0になったあとはRVRで設定された値になります。つまり、RVRの値が割り込みの周期ということになります。
さて、このCVRがどのくらい時間で値が減っていくかですが、CALIBレジスタにその値が記されています。
CALIBには10ミリ秒ごとにどの程度値が減るかの値が設定されるリードオンリーレジスタです。この下位24ビットの値を見てあげれば、RVRやCVRを適切に設定できます。
これらのレジスタがどこにマップされているかもこのリファレンスマニュアルのB3.3.2章に書かれています。

さて、あとはこれらのレジスタを使うだけです。これらのメモリアドレスにアクセスするには標準ライブラリの`core::ptr`内にある[`read_volatile`](https://doc.rust-lang.org/core/ptr/fn.read_volatile.html)と[`write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html)を使います。
volatileというのは揮発性のという意味で、C言語などでは変数の修飾子としてつけることで、その変数へのアクセスの最適化をさせないようにすることができます。
同様に`read_volatile`や`write_volatile`も最適化によって変更されてほしくないアクセス、つまり今回のようなメモリマップドIOに使える関数です。
今回は1秒毎に割り込みが発生するように設定してみましょう。`systick.rs`内の`init`関数を以下のように書き換えましょう。
```
use cortex_m_semihosting::hprintln;
use core::ptr::{read_volatile, write_volatile};

const CSR_ADDR: usize = 0xE000_E010;
const RVR_ADDR: usize = 0xE000_E014;
const CVR_ADDR: usize = 0xE000_E018;
const CALIB_ADDR: usize = 0xE000_E01C;

pub fn init() {
    hprintln!("Systick init").unwrap();
    unsafe {
        write_volatile(CVR_ADDR as *mut u32, 0);
        let calib_val = read_volatile(CALIB_ADDR as *const u32) & 0x00FF_FFFF;
        write_volatile(RVR_ADDR as *mut u32, calib_val * 100);
        write_volatile(CSR_ADDR as *mut u32, 0x3);
    }
}
```
`read_volative`及び`write_volatile`はともに`unsafe`な関数なので`unsafe`ブロックで囲う必要があります。
なお、アドレスを渡す際に`usize`型をポインタにキャストしている部分がありますが、この操作自体はRustでは安全とされています。
ただし、このアドレスをいざ読み書きする段階になると、この先にあるアドレスがちゃんとしたものになっているかの保証もないですし、ライフタイムのチェックもないので`unsafe`となるわけです。

さて、これでコンパイルすると、今までのメッセージに加えてSysTickハンドラが1秒毎に呼び出されてメッセージが表示されているはずです。
また、デバッガで動作を止めると、`Reset`関数自体は最後の無限ループで止まっているのがわかります。
