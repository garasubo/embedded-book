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
rustup install nightly-2019-09-18
```


## 使用ボード
STマイクロエレクトロニクス社の[NUCLEO-F429ZIボード](https://www.st.com/ja/evaluation-tools/nucleo-f429zi.html)を使用します。
ただし、この本では多くのペリフェラルは使わないので、ST社のCortex-Mの他のボードでも十分に再現可能かと思われます。

## 仕様書・ドキュメント
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

- [Nucleo-144ボードのユーザーマニュアル ()](https://www.st.com/content/st_com/ja/products/evaluation-tools/product-evaluation-tools/mcu-mpu-eval-tools/stm32-mcu-mpu-eval-tools/stm32-nucleo-boards/nucleo-f429zi.html)
