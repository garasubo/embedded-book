# 環境構築
筆者はUbuntu 18.04で動作確認をしていますが、基本的にはWindowsやMac環境でも構築可能です。

## Rust
`rustup`を使う方法が公式でも全プラットフォーム共通で推薦されているので、これに従います。
[https://rustup.rs/]の方法に従って、rustupをインストールします。LinuxやMacのようなUnixライクなOSの場合、以下のようにインストールします。
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

rustupはRustコンパイラのインストール・アップデートやバージョン管理をしてくれるツールです。
今回は特に`nightly`と呼ばれる、いわゆるベータ版のようなRustコンパイラを使うことになるのですが、nightlyと安定版である`stable`の切り替えもコマンド一つで行えて大変便利です。

今回はnightlyの2019-09-19のバージョンを使います。以下のようにして`nightly-2019-09-18`をインストールしましょう。
```
$ rustup install nightly-2019-09-18
```

さらにArmのクロスコンパイル用の環境を入れるために以下のコマンドを実行します
```
$ rustup target add thumbv7em-none-eabihf
```

## 使用ボード
STマイクロエレクトロニクス社の[NUCLEO-F429ZIボード](https://www.st.com/ja/evaluation-tools/nucleo-f429zi.html)を使用します。
ただし、この本では多くのペリフェラルは使わないので、ST社のCortex-Mの他のボードでも十分に再現可能かと思われます。

## 仕様書・ドキュメント
### ハードウェア関連
低レイヤーシステムは仕様書をちゃんと読み理解する、ということがとても重要になります。
マイコンボードの種類によっては、仕様書を手に入れるためには面倒な契約を結ぶ必要があるケースもありますが、今回使うNucleoボード及びそのCPUの仕様書はすべてオンラインで入手可能です。
この本では、実際に仕様書を参照しつつ、OSを書いていきますのでこれらもダウンロードしてください。
なお、以下のリンクにある仕様書は基本的に英語版です。日本語版もある場合がありますが、最新版ではないため英語版を参照するほうが安全です。

- [Arm®v7-M Architecture Reference Manual](https://developer.arm.com/docs/ddi0403/ed/armv7-m-architecture-reference-manual)
ボードに乗っているCPUの基本的な仕様書です。アセンブラ命令の解説や、割り込みが発生したときの挙動、システムレジスタの仕様などが書かれています。

- [ARM Cortex-M4 Processor Technical Reference Manual Revision r0p1 Documentation](https://developer.arm.com/docs/100166/0001)
Arm v7-Mの仕様に基づいたアーキテクチャであるCortex-M4のマニュアルです。[日本語版](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0439cj/index.html)も多少古いバージョンですが存在します。
割り込みコントローラやメモリ保護機構などの解説があります。また、Arm v7-Mについても簡単な説明があります。

- [STM32F42xxxのリファレンスマニュアル](https://www.stmcu.jp/design/document/reference_manual/51544/)
今回使うボードのCPUのペリフェラル（周辺機器）の仕様書です。例えばLEDを光らせるにはGPIOというペリフェラルから信号を送るのですが、そういうものの使い方が知りたい場合はこのマニュアルを基本的に読むことになります。

- [STM32F429xxのデータシート](https://www.st.com/resource/en/datasheet/stm32f429zi.pdf)
CPUについての詳細です。CPUの仕様はリファレンスマニュアルだけでなく、こちらのデータシートも併せて読むことになります。


- [Nucleo-144ボードのユーザーマニュアル](https://www.st.com/content/st_com/ja/products/evaluation-tools/product-evaluation-tools/mcu-mpu-eval-tools/stm32-mcu-mpu-eval-tools/stm32-nucleo-boards/nucleo-f429zi.html)
ボードそのものについてのドキュメントです。どの端子やピンがどうCPUに結びついているかなどの使い方を調べるために使います。

### Rust関連
Rustそのものについてのドキュメントもしっかりと見ていく必要があります。
まず、Rustに関する知識が一切ない場合、TRPLと呼ばれるチュートリアルを一通り眺めてみるのが良いでしょう。
- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [日本語版](https://doc.rust-jp.rs/book/second-edition/)
本書ではRustに関する初歩的な構文の知識などには触れないため、もしわからないことがあればこちらを参照してください。

低レイヤーのプログラミングを行うにあたって、Rustの詳細な実装を知る必要がある場面があります。
そのための本としてThe Rustonomiconというのがあります。
- [Ther Rustonomicon](https://doc.rust-lang.org/nomicon/index.html)
こちらは必要になったら参照することになりますが、すべてを理解しておく必要はありません（正直なところ筆者もすべての項目を把握していないです）。
