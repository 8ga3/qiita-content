---
title: M5StackのUnit Encoderのファームウェア・アップデート
tags:
  - M5stack
  - UnitEncoder
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
<img width="360" alt="UnitEncoder1" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d752bc0c-94a7-4669-0256-c723d09ac2a1.jpeg">

M5StackにI2Cで接続して使うUnit Encoderを購入しました。

[商品ページ](https://docs.m5stack.com/en/unit/encoder)によると、どうやらファームウェアを書き換えることができるとあります。それならばと書き換えてにチャレンジしてみました。


https://ssci.to/7987

# ファームウェア

[GitHub](https://github.com/m5stack/M5UnitEncoder_Firmware)に、説明とバイナリファイルがおいてあります。基本的にこのとおり進めていきました。I2Cプロトコルマップをみると、MODEとRESETが追加されているとあります。カウンターの値を書き込めるのが、地味に嬉しい追加機能です。

[V1.1のファームウェア](https://github.com/m5stack/M5UnitEncoder_Firmware/blob/main/M5UnitEncoder_Firmware_v1.1.hex)をダウンロードします。


https://github.com/m5stack/M5UnitEncoder_Firmware

# Unit Encoder分解

分解といってもネジひとつはずすだけです。基板にSTM32F030が実装されています。写真上部のスルーホールがフラッシュするためのピンに繋がっているので、ここにライターを接続します。

<img width="540" alt="UnitEncoder2" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/680d3a3d-9ac1-94ae-f28b-d014826aabfe.jpeg">

# ライター

本来であればSTMicroelectronics社のSTLINKを使って書き換えます。まあまあの値段でして、私は所有していません。そこでどうするかというと、安いコンパチのSTLINK/V2を使用します。数百円程度で手に入ります。

<img width="360" alt="UnitEncoder3" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/28a95516-ef2d-159f-0c52-243d63e31f1f.jpeg">

検索したら、いろいろなところで売ってますね。信頼できるのかどうかはご自身で判断してください。

* https://www.amazon.co.jp/dp/B07JKZ4FKX
* https://ja.aliexpress.com/item/1005004997698975.html

# フラッシュ

フラッシュには[STM32CubeProg](https://www.st.com/ja/development-tools/stm32cubeprog.html)が必要です。STLINKとUnit Encoderを表を参考に繋いでください。まちがえて3.3Vを5Vにしがちですので注意しましょう。

| STLINK   | Encoder |
|:---------|:-------:|
|(1) RST   | R       |
|(2) SWCLK | C       |
|(4) SWDIO | D       |
|(6) GND   | G       |
|(8) 3.3V  | V       |

ジャンパーワイヤーを使いますが、スルーホールにはんだ付けはしたくありません。バラバラのワイヤーをまとめるため、ピンソケットに挿してまとめました。マスキングテープで留めてもいいかもしれません。

<img width="360" alt="UnitEncoder4" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f9ffa9de-675b-387c-890c-894ea9b48057.jpeg">

気合で手でおさえます。このままGitHubの説明に従ってSTM32CubeProgで書き込みます。

<img width="360" alt="UnitEncoder5" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e2536503-a8ee-0305-7e3a-09073a79e83e.jpeg">

基板を手でおさえながら、STLINKをPCに接続し、アプリを操作してフラッシュ。これがなかなかうまくいかず、10回目くらいで成功しました😅

# I2Cレジスタマップ

GitHubの説明にひとつだけ追加しています。それはRGBのLED NUMを0にすると、２つのLEDをまとめてセットできます。exampleのソースコードには書いてあるのに、なぜかドキュメントには書いてありません。

| OPERATION     | REG MAP | 0 | 1 | 2 | 3 |
|:--------------|:-------:|:--------|:--------|:-------|:--------|
| MODE (W)      | 0x00    | 0=PULSE<br>1=AB | | | |
| COUNTER (R/W) | 0x01    | COUNTER VALUE<br>L | <br>H | |
| SWITCH (R)    | 0x02    | SWITCH<br>0 or 1 | | | |
| RGB (W)       | 0x03    | LED NUM<br>0, 1, 2 | R | G | B |
| RESET         | 0x04    | 1 >= RESET<br>COUNTER | | | |
