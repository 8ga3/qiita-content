---
title: ジンバルモーター、SimpleFOCで動かす(1)
tags:
  - BLDC
  - Arduino
  - ArduinoUnoR4
  - SimpleFOC
  - PlatformIO
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

バイラテラル制御、ハプティクスの学習を目的として制作しました。その過程を記録します。

# 開発環境

* macOS Sonoma 14.5 (Intel & Apple silicon)
* PlatformIO Core 6.1.15
* PlatformIO Home 3.4.4
* Simple FOC 2.3.3
* Visual Studio Code 1.90.2

# パーツリスト

* Arduino Uno R4 Minima
* MARSPOWER Brushless Gimbal Motor 2208 80T
* SimpleFOCMini V1.0 (DRV8313) (公式のものではなくAliexpressで売っている製品)
* 磁気エンコーダ AS5600
* YAHATA スペーサー 5x15 ニッケルめっき
* バッテリー LiPo 3Cell 800mAh 65C

# 接続図

![BLDC_SimpleFOC_AS5600.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/fc481c46-e237-f3de-40cb-9ae2ba4667e7.png)
 
# BLDCへ磁気エンコーダを固定

磁気エンコーダ用の磁石をジンバルモーターに接続します。
磁石はこんな感じだと思います。同じ形の磁石でも磁化方向はいろいろあるようですので、付属の磁石を使うか仕様をよく確認します。

![mag_NS.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/9bd6a8e8-f584-c1fa-b611-38187255cc39.png)

1. ジンバルモーターの下側、シャフトが1mmほど出っ張っています。ここがローターと一緒に回ります。回転するローターのことをベルと言うこともあります。
2. シャフトの終端に磁気エンコーダが読み取るための磁石を固定します。
3. シャフトと磁石を固定するジョイントを設置します。
4. 外側に防磁のために鉄のリングで囲ってあげます。

![mag1-4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/dee724c5-4cbe-2341-5e33-2efb13396b24.png)

防磁はホームセンターでスペーサーを買ってきて、2mmの高さになるように金鋸で切り出しました。

当初、防磁しないでモーターを動かしたところ、磁気エンコーダが値を読み取らなくなりました。

![IMG_6689.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b61245ec-8e2e-46d0-6dba-fd17ebfbbc93.jpeg)

ジョイントは3Dプリンタで作りました。こんな感じで接続しました。磁石と防磁リングはちょっと緩かったのですが、磁力でくっついてくれました。回転速度によっては接着したほうがいいかもしれません。

![IMG_6669.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f53592da-54cf-4f2c-0b8d-71d07a61b68c.jpeg)

モーターに台座をネジ止めします。磁石と磁気エンコーダを1mm弱くらいの距離で設置しました。できれば、AS5600をI2Cでステータス・レジスタ(0x0B)を読んで確認します。磁力が強すぎ(BIT3)・弱すぎ(BIT4)だとビットが立ちます。Bit5が立っていれば正常です。（詳細はマニュアルを参照してください。）

![mag5-6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/508e20f4-13b3-1330-5038-7d93da68b2df.png)

横から見るとこんな感じになります。ネジが邪魔で磁石がちょっとしかみえませんね…

![mag7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/8dec114c-1ce4-5917-72c5-f1f9f33c13f6.png)

# 組付け

CGだとこんな感じになります。

![mag8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/acfbe534-d3a1-4a02-0e15-184c89e0924b.png)

動きがわかるようにステーをつけてます。

![IMG_6679.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a789449c-13b0-60dd-7d4c-85b1bdf0c1ec.jpeg)

サイドビュー

![IMG_6685.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/6ec2da53-0405-99bf-41c7-0db7da28fb38.jpeg)

SimpleFOCMiniにピンヘッダーをつけ、それぞれケーブルで接続したところです。

![IMG_6687.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ffe17b4a-cd22-f703-f07a-e102e49138a8.jpeg)

# 失敗談

当初、ドローン用ブラシレスモーターで動かそうとしていました。しかし、全く動きませんでした。

なぜ動かないか。ドローン用ブラシレスモーターは高速回転のためにホルマル線のターン数（巻数）を10〜20回くらいにしているようです。ターン数が少ないと高回転・低トルク・高電流になるそうです。SimpleFOCMiniはジンバルモーターを動かす目的なので、最大電流は3Aということで動かなかったわけです。

なおドローン用のESCは、高電流に耐えられるMOSFETが使われています。

そのあと購入した、今回使っているブラシレスモーターは、80ターンの低速・高トルクタイプです。

![IMG_6691.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e9352401-f2dc-ab11-4fe0-d06910a1e3b5.jpeg)

# おわりに

モジュールをつなぐところまでいきました。

次回、コーディングしていきます。
