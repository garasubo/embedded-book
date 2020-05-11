# スケジューラを実装する
前回の章でアプリケーションプロセスへの切り替えを実装しました。
しかし、現状だと再びアプリケーションに突入したり、複数のアプリケーションを動かしたり、といったことができていません。
この章では、簡単なラウンドロビン型のスケジューラを実装して複数のアプリケーションを動かすことを目標としましょう。

スケジューラを実装するには前章まででやってきたアプリケーションプロセスへの切り替えに加えて以下のことが必要です。
* 次に実行可能なプロセスを指すための構造体を実装する
* プロセスの状態をメモリ上に保管する構造体とその保存ロジックの実装

順番にやっていきましょう

## 連結リストを実装する
スケジューラを実装するに当たって、実行中のプロセスの情報を連結リストの形式で持っておきたいです。
今回実装するスケジューラでは、実行が終わったプロセスを末尾に追加して、先頭から次に実行するプロセスを取ってくるため連結リストを利用したいです。
しかしながら、`no_std`のプログラミングではお馴染みの動的配列である`Vec`や`LinkedList`は使えないので似たような構造体がほしいです。

では、連結リストを`linked_list.rs`内に実装してみましょう。
C言語などで実装したことがある人がいるとは思いますが、普通はおおよそこんな感じになるでしょう。
* リスト内に入る各アイテムは、そのアイテムの実際の値と次のアイテムへのポインタを持つ
* リスト構造体そのものは先頭要素へのポインタと末尾要素へのポインタを持つ
* 新しいアイテムを追加するときは現在の末尾要素の次に追加して、リストの持つ末尾要素ポインタを更新
* 先頭から要素を取り出すには先頭要素ポインタから取り出して、次の要素をリストの持つ先頭要素ポインタに更新

Rustでは生ポインタもありますが、通常であればポインタよりも参照を用いた実装が好まれます。
しかし、先程あげたような機能を持つ連結リストは参照では実装は困難です。実際に見てみましょう。

まず、リストそのものと、リストに格納される各要素の定義は以下のような感じになるでしょう。
```
pub struct ListItem<'a, T> {
    value: T,
    next: Option<&'a mut ListItem<'a, T>>,
}

pub struct LinkedList<'a, T> {
    head: Option<&'a mut ListItem<'a, T>>,
    last: Option<&'a mut ListItem<'a, T>>,
}
```

試しに末尾に新しい要素を追加するというメソッドを実装してみましょう。素直に実装するとこんな感じでしょうか。
```
impl<'a, T> LinkedList<'a, T> {
    pub fn push(&mut self, item: &'a mut ListItem<'a, T>) {
        if self.last.is_none() {
            self.last = Some(item);
            self.head = Some(item);
        } else {
            let prev_last = self.last.replace(item);
            prev_last.map(|i| i.next = Some(item));
        }
    }
}
```

しかし、これはコンパイルエラーになります。なぜなら、`item`はミュータブルな参照なのにもかかわらず、
`self.last`と`self.head`ないし1つ手前の`ListItem`の`next`の2つのメンバーに保持される必要があります。
ミュータブルな参照は複数存在できないので、このような形では素直には実装できないです。

では、標準ライブラリではどのように実装されているかというと、生ポインタを使うことで解決しています。
確かに生ポインタを使うことは安全ではないのですが、標準ライブラリ内部ではしばしば`unsafe`な部分が存在しています。
その代わり提供するインターフェスは生ポインタを使うことがないようにして安全でない部分を限定している、というわけです。
そういうことで、ここでは生ポインタを使うことにしましょう。その代わり、このモジュールにはテストを書いておき、きちんとバグがないよう確認しておきましょう。

まず、テストとしては以下のようなものを書いておきましょう。
```
#[cfg(test)]
mod test {
    use ListItem;
    use LinkedList;

    #[test]
    fn test_list() {
        let mut item1 = ListItem::new(1);
        let mut item2 = ListItem::new(2);
        let mut item3 = ListItem::new(3);
        let mut list = LinkedList::new();

        list.push(&mut item1);
        list.push(&mut item2);
        list.push(&mut item3);

        assert_eq!(Some(&mut 1), list.head_mut());
        let result1: &u32 = list.pop().unwrap();
        assert_eq!(Some(&mut 2), list.head_mut());
        let result2: &u32 = list.pop().unwrap();
        assert_eq!(Some(&mut 3), list.head_mut());
        let result3: &u32 = list.pop().unwrap();
        assert_eq!(1, *result1);
        assert_eq!(2, *result2);
        assert_eq!(3, *result3);

        assert!(list.is_empty());

        let mut item4 = ListItem::new(4);
        let mut item5 = ListItem::new(5);
        list.push(&mut item4);
        list.push(&mut item5);

        let result4: &u32 = list.pop().unwrap();
        let result5: &u32 = list.pop().unwrap();
        assert_eq!(4, *result4);
        assert_eq!(5, *result5);

        assert!(list.is_empty());
    }
}

```

最低限のテストではありますが、とりあえず必要なものは見えてきたと思います。
* `push`：リストの末尾に要素を追加する
* `pop`：先頭要素を取り出す。返り値は`Option`でラップしてリストが空なら`None`が返る
* `head_mut`： 先頭要素のミュータブルな参照を`Option`でラップして返す。リストそのものは変更しない。
* `is_empty`：リストの中に要素が存在するかどうか

`list.pop().unwrap()`の結果を`&u32`として変数に束縛していますが、これは`ListItem<'a, T>`に対して[`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html)を実装していることを想定しています。
以降、生ポインタを使っての実際の実装を見せていきますが、ある程度すでにRustに習熟しているのであれば、この仕様を満たすように自力で実装するのもいいでしょう。
以下に示すのはあくまで一例です。また、実装の踏み込んだ解説はあまりしないでおきます。

まずは構造体そのもの定義を変えましょう。参照はすべてポインタに置き換えてしまうのが一番愚直なやりかただと思うので、そうします（もちろん一部を参照を残すこともできます）。

```
use core::ptr::NonNull;
use core::marker::PhantomData;

pub struct ListItem<'a, T> {
    value: T,
    next: Option<NonNull<ListItem<'a, T>>>,
    marker: PhantomData<&'a ListItem<'a, T>>,
}

pub struct LinkedList<'a, T> {
    head: Option<NonNull<ListItem<'a, T>>>,
    last: Option<NonNull<ListItem<'a, T>>>,
    marker: PhantomData<&'a ListItem<'a, T>>,
}
```
[`NonNull`](https://doc.rust-lang.org/core/ptr/struct.NonNull.html)と[`PhantomData`](https://doc.rust-lang.org/core/marker/struct.PhantomData.html)という構造体を使っています。
`NonNull`はnon-nullでないポインタを意味します。普通の`*mut T`に対して0ではないという制約がついているわけですが、これにより`Option`でくくったときに0という値を`None`として使うようにコンパイルされるため、`Option<NonNull<T>>`と`*mut T`のコンパイル後のサイズが同じという利点が生まれます。
すべてをポインタに置き換えてしまうとライフタイムの情報がなくなってしまいます。コンパイル時にこれらの情報を失わないように`PhantomData`を使ってます。
これはコンパイルされるとサイズが全くないものになるのですが、実際のメンバーのように振る舞うことで型引数をさも要求しているようにコンパイラに見せかけるための構造体です。
詳しくは公式ドキュメントを参照してください。

関数の実装は以下のようにしてみました。
```
impl<'a, T> ListItem<'a, T> {
    pub fn new(value: T) -> Self {
        ListItem {
            value,
            next: None,
            marker: PhantomData,
        }
    }
}

impl<'a, T> LinkedList<'a, T> {
    pub fn new() -> Self {
        LinkedList {
            head: None,
            last: None,
            marker: PhantomData,
        }
    }

    pub fn push(&mut self, item: &'a mut ListItem<'a, T>) {
        let ptr = unsafe { NonNull::new_unchecked(item as *mut ListItem<T>) };
        let prev_last = self.last.replace(ptr);

        if prev_last.is_none() {
            self.head = Some(ptr);
        } else {
            prev_last.map(|mut i| unsafe {
                i.as_mut().next = Some(ptr);
            });
        }
    }

    pub fn is_empty(&self) -> bool {
        self.head.is_none()
    }

    pub fn head_mut(&mut self)-> Option<&mut T> {
        self.head.map(|ptr| unsafe { &mut *ptr.as_ptr() }.deref_mut())
    }

    pub fn pop(&mut self) -> Option<&'a mut ListItem<'a, T>> {
        let result = self.head.take();
        let next = result.and_then(|mut ptr| unsafe {
            ptr.as_mut().next
        });

        if next.is_none() {
            self.last = None;
        }

        self.head = next;

        result.map(|ptr| unsafe { &mut *ptr.as_ptr() })
    }
}
```
`pop`の返り値が`T`の参照でなく`ListItem`の参照なのは`pop`で返ってきた構造体を再度リストに追加する、ということができるようにするためです。
`push`が`T`でなく`ListItem`の参照を引数に取っているのは少々美しくないようにも思えます。しかし、現状動的なメモリ確保の方法を実装していません。
そのため、`ListItem`のためのメモリ領域を確保することができないのでほかから渡してもらう必要があるというわけです。

構造体のメンバーをすべてプライベートにしてしまっているので、`ListItem`からvalueの値を取り出すことがこのままではできません。
新たなメソッドを追加するのもいいですが、`Deref`を実装するという方法で実現してみましょう。

```
use core::ops::{Deref, DerefMut};

impl<'a, T> Deref for ListItem<'a, T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.value
    }
}

impl<'a, T> DerefMut for ListItem<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.value
    }
}
```


ここまで書いたところでテストを走らせたいのですが、このプロジェクト全体が標準ライブラリなしでArmへのクロスコンパイルを前提として書かれていることもあり、このまま`cargo test`を実行しても正常に動きません。
しかし、このモジュール単体はアーキテクチャへの依存がないため、このプロジェクトは切り離してのテストが可能です。
手軽な方法としては`rustc --test`でこのファイルのみをテストしてしまうことでしょうか。その際、このファイルの冒頭に`#![no_std]`をつける必要があります。
ただし、モジュールに本来クレートレベルのアトリビュートである`no_std`をつけてしまうと、プロジェクト全体をコンパイルしたときに警告が出てしまいます。
他の方法としては、これを別クレートとして新しいプロジェクトに切り出してしまうことです。これならば、その新しくつくったプロジェクト上で`cargo test`を実行すればいいです。
この方法ならば、コンパイル時の警告もないので、よりまっとうな方法になるでしょう。

## プロセスの状態を保存する
次にプロセス状態を保存する構造体を定義しましょう。
プロセスの状態としてまず、そのプロセスで使うスタックポインタは保存しなければなりません。
さらに保存しなければならないのは、スタックに退避されないレジスタたちです。

スタックポインタの値はusizeとして、退避されていないレジスタはまとめて`u32`型の配列として持っておきましょう。
```
pub struct Process<'a> {
    sp: usize,
    regs: [u32; 8],
    marker: PhantomData<&'a u8>,
}
```
プロセスをつくる関数もつくりましょう。スタック用の領域とプロセスの中で実行したい関数を渡したらプロセス構造体が返ってくるとよさそうですね。
その際、スタック領域の初期化処理もしてしまいましょう。

```
impl<'a> Process<'a> {
    pub fn new(stack: &'a mut [u8], app_main: extern "C" fn() -> !) -> Self {
        let sp = (&stack[0] as *const u8 as usize) + stack.len() - 0x20;
        let context_freame: &mut ContextFrame = unsafe { &mut *(sp as *mut ContextFrame) };
        context_freame.r0 = 0;
        context_freame.r1 = 0;
        context_freame.r2 = 0;
        context_freame.r3 = 0;
        context_freame.r12 = 0;
        context_freame.lr = 0;
        context_freame.return_addr = app_main as u32;
        context_freame.xpsr = 0x0100_0000;

        Process {
            sp: sp as *mut u8,
            regs: [0; 8],
            marker: PhantomData,
        }
    }
}
```

このプロセスを実行させるコードも実装しましょう。
プロセスは割り込みが発生すると中断する可能性があります。それを考慮して実装する必要があります。
前章で実装したプロセススタックポインタの書き込みの他に、`ContextFrame`の中に保存されていないレジスタの書き戻しをする必要があります。
これは中断されたプロセスを再開するためのものです。
また、カーネルに復帰した際、その時点でのプロセススタックポインタと`ContextFrame`の中に保存されていないレジスタを保存する処理も必要です。

```
    pub fn exec(&mut self) {
        unsafe {
            asm!(
                "
                msr psp, r0
                ldmia r1, {r4-r11}
                svc 0
                stmia r1, {r4-r11}
                mrs r0, psp
                "
                :"={r0}"(self.sp)
                :"{r0}"(self.sp), "{r1}"(&self.regs)
                :"r4", "r5", "r6", "r7", "r8", "r9", "r10", "r11"
                :"volatile"
            );
        }
    }
```

`ldmaia`と`stmia`は複数のレジスタの値をまとめてメモリからロード・ストアするための命令です。
スタックポインタ上にレジスタを退避させたりするのに使われます。`r1`がメモリアドレスになり、この命令後に本来は書き換えられるのですが、
`ldmai`後に`stmia`を実行しているので変化した分がすぐに戻されているため、変化が打ち消されているため値の変化はトータルではありません。
`mrs`が`msr`の逆でシステムレジスタを読み出すための命令です。

ここまでできたら、前回実装したプロセス切り替え部分のコードをこれらのコードで置き換えてみましょう。
`APP_STACK`を定義したあとのコードを以下のように書き換えてみましょう。

```
    #[link_section = ".app_stack"]
    static mut APP_STACK: [u8; 2048] = [0; 2048];

    let mut process = Process::new(&mut APP_STACK, app_main);
    process.exec();

    hprintln!("Kernel").unwrap();
```

ここまでできたら一度コンパイルしてみて動作確認してみましょう。

## スケジューラ本体の実装
必要なパーツは全て揃ったところで、これらを組み合わせてスケジューラを実装しましょう。
今回実装するのは登録されたプロセスを順番に実行していくだけのラウンドロビン型のスケジューラです。
なお、プロセスが終了してメモリが開放されることは今回は考えないことにします。
`scheduler.rs`の中に実装していきましょう。

まず、スケジューラの構造体のメンバーとして保持しておきたいのは実行したいプロセスの連結リストです。
```
use crate::linked_list::{LinkedList, ListItem};
use crate::process::Process;

pub struct Scheduler<'a> {
    list: LinkedList<'a, Process<'a>>,
}
```

実装する関数としては、コンストラクタの他にプロセスの追加関数と、スケジューラ内の関数を実行し続ける関数を実装しましょう。
```
impl<'a> Scheduler<'a> {
    pub fn new() -> Self {
        Scheduler {
            list: LinkedList::new(),
        }
    }

    pub fn push(&mut self, item: &'a mut ListItem<'a, Process<'a>>) {
        self.list.push(item);
    }

    fn schedule_next(&mut self) {
        let current = self.list.pop().unwrap();
        self.list.push(current);
    }

    pub fn exec(&mut self) -> ! {
        loop {
            let current = self.list.head_mut();
            if current.is_none() {
                unimplemented!();
            }
            current.map(|p| {
                p.exec();
            });
            self.schedule_next();
        }

    }
}
```

`exec`が実行用の関数です。リストの先頭要素を実行して、実行が中断されるとリストから一度取り出され、末尾に再度追加されます。
各プロセスが実行を中断するには現状はプロセスが自発的に`svc`命令を発行するしかありません。
本来であれば、OSが何らかの例外が発生したら切り替えるなどの処理を実装することが多いです（例えばタイマー割り込みで一定時間以上実行していたら切り替える）。が、今回は何もしないことにしましょう。

では、このスケジューラの動作確認のために、簡単なプロセスを３つ用意して、それらをスケジューリングしてみましょう。
まず、各プロセス用の関数ですが、`app_main`と同じように適当な文字を出力させる関数を追加で２つ用意しましょう。
ただし、プロセスが再開したあとなにもしない無限ループを実行されるとなにもわからなくなるので、文字出力と`svc`をひたすら繰り返す、というものにしましょう。
```
extern "C" fn app_main2() -> ! {
    loop {
        hprintln!("App2").unwrap();
        unsafe { asm!("svc 0"::::"volatile"); }
    }
}

extern "C" fn app_main3() -> ! {
    loop {
        hprintln!("App3").unwrap();
        unsafe { asm!("svc 0"::::"volatile"); }
    }
}
```

また、`app_main`も以下のように書き換えましょう。
```
extern "C" fn app_main() -> ! {
    let mut i = 0;
    loop {
        hprintln!("App: {}", i).unwrap();
        unsafe { asm!("svc 0"::::"volatile"); }
        i += 1;
    }
}
```

あとは、`Reset`関数の末尾を書き換えてスケジューラにプロセス３つを追加しましょう。
まずは、モジュールから使うものを以下のように宣言しておきます。
```
mod linked_list;
use linked_list::ListItem;
use process::Process;
mod scheduler;
use scheduler::Scheduler;
```

あとは、`Process`を`ListItem`でラップしたものを3つつくり、これをスケジューラの中に入れて実行するだけです。
なお、`Scheduler::exec`自身が発散する関数なので、最後の無限ループも取り除くことができます。
```
    #[link_section = ".app_stack"]
    static mut APP_STACK: [u8; 2048] = [0; 2048];
    #[link_section = ".app_stack"]
    static mut APP_STACK2: [u8; 2048] = [0; 2048];
    #[link_section = ".app_stack"]
    static mut APP_STACK3: [u8; 2048] = [0; 2048];

    let mut process1 = Process::new(&mut APP_STACK, app_main);
    let mut item1 = ListItem::new(process1);
    let mut process2 = Process::new(&mut APP_STACK2, app_main2);
    let mut item2 = ListItem::new(process2);
    let mut process3 = Process::new(&mut APP_STACK3, app_main3);
    let mut item3 = ListItem::new(process3);
    let mut sched = Scheduler::new();
    sched.push(&mut item1);
    sched.push(&mut item2);
    sched.push(&mut item3);

    sched.exec();
}
```

こうしてビルドすれば以下のような実行結果が得られるはずです。
```
Hello World
Systick init
App: 0
App2
App3
App: 1
App2
App3
App: 2
App2
App3
App: 3
App2
...
```

なお、Systick割り込みがなかなか表示されないことに気がつくと思いますが、これはSystickのタイマーがデバッグ状態では値が減らないためであり、
デバッグ状態を利用するセミホスティングを利用した出力をたくさんしていると現実の時間が進んでいても、Systickのタイマーが減らないという事態が起きているからです。
試しに`hprintln`命令を取り除いてあげるとちゃんとSystickが呼び出されるようになるはずです。
