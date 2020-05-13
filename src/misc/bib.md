このドキュメントを執筆するにあたってハードウェアのマニュアル以外に参考にしたもの、
参考にはしていないが、この先自作OSをつくりこむにあたって参考になるであろう書籍やドキュメントを紹介します。

- [The Embednomicon](https://docs.rust-embedded.org/embedonomicon/preface.html)
Rust Embeddedワーキンググループが公開しているRustでベアメタルプログラミングをするための解説ドキュメントです。
このドキュメントもこのドキュメントがベースになっています。より踏み込んだロギングやDMAのためのテクニックも解説しています。

- [The Embedded Rust Book](https://docs.rust-embedded.org/book/index.html)
こちらもRust Embeddedワーキンググループが公開しているRustでベアメタルプログラミングをするための解説ドキュメントですが、Rust Embeddedワーキンググループが用意したクレートを活用してのプログラミングです。
今回はあえてこれらのクレートは使わないような実装にしましたが、特にペリフェラルのドライバを書くのに参考になると思います。

- Andrew S. Tanenbaum、Herbert Bos 「Modern Operating Systems: Global Edition」
マイクロカーネルで有名なタネンバウム先生らが書いたOSの有名な教科書です。古いエディションであれば日本語版もあります。
かなり量も多いのですが、OSの基本的な概念を幅広く網羅されています。

- [Writing an OS in Rust](https://os.phil-opp.com/)
x86系のプロセッサを対象にしたものですが、RustでOSを書くチュートリアルになっています。
x86系固有の知識以外にも、エミュレータを組み合わせた統合テストの書き方や動的なメモリ確保のアルゴリズムの実装なども解説されているので、それ以外のプロセッサでも参考になります。

- [組込み/ベアメタルRustクックブック](https://booth.pm/ja/items/1478032)
Rustで組込みプログラミングをするのに有用なテクニックが幅広く紹介されています。日本語のドキュメントです。
