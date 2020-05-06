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

        let result1: &u32 = list.pop().unwrap();
        let result2: &u32 = list.pop().unwrap();
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
TBD