---
title: ジンバルモーター、SimpleFOCで動かす(8) - BLDCモーター2個接続
tags:
  - Arduino
  - BLDC
  - SimpleFOC
  - ArduinoUnoR4
private: false
updated_at: '2024-08-04T23:12:35+09:00'
id: 5f4be147a180e6b8fb06
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/29c241aa43adbf6908d1)の続きです。

7回目で終わるつもりでしたが、記録しておきたい事象があったのでもう少し続けます。

Arduino Uno R4にBLDCモーターを2個接続してみます。

# 接続図

これまでモーターがひとつではこのように接続していました。（電流センサは一旦外しています。）

![BLDC_SimpleFOC_AS5600.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/fc481c46-e237-f3de-40cb-9ae2ba4667e7.png)

２個接続する場合はこのようになります。

![BLDC_SimpleFOC_BLDCx2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/10f2b776-3909-6b4d-901a-d5930300902b.png)

今回はArduino Uno R4とモータードライバとの接続について書きます。I2Cの磁気エンコーダについては次回になります。

# Arduino Uno R4のタイマ

モータードライバへの信号出力にPWMをつかっています。Arduinoのピンソケットの番号に **〜** 記号がついている場所が、PWM対応となってます。

しかし、R4になってからプロセッサがRenesas RA4M1になったため、実際には **〜** 記号がないピンソケットでもPWMが使えます。

**Renesas RA4M1グループ ユーザーズマニュアル ハードウェア編** を確認します。PWMにはタイマが必要です。SimpleFOCはAGTとWDTは使えません。GPT32とGPT16が使用できます。

* 32 ビット汎用 PWM タイマ(GPT32)× 2チャンネル
* 16 ビット汎用 PWM タイマ(GPT16)× 6チャンネル
* 低消費電力非同期汎用タイマ(AGT)× 2チャンネル
* ウォッチドッグタイマ(WDT)

GPTには出力がAとBの2個あります。ここで使用しているSimpleFOCMiniはPWM x3で制御しますが、AとBを同時に使用不可の制限があります。（PWM x6のモータードライバの場合はハイサイドとローサイドを一組にして同じPWMのAとBをつなぐことで、デッドタイムをハードウェアで自動的に管理してくれるようです。）

# Arduino Uno R4とモータードライバSimpleFOCMiniとの接続

Arduino Uno R4のソケットが、具体的にRA4M1のどこにつながっているか調べてみました。下の表になります。デジタルのソケットはMinimaとWi-Fiではかなり違いました。アナログは同一でした。

|Arduino<br>Socket|Minima<br>I/O Port|Minima<br>Channel|Minima<br>Output|Wifi<br>I/O Port|Wifi<br>Channel|Wifi<br>Output|Motor<br>Driver|
|:--:|:----:|:-:|:-:|:----:|:-:|:-:|:-:|
|D0  | P301 | 4 | B | P301 | 4 | B |
|D1  | P302 | 4 | A | P302 | 4 | A | 2 |
|D2  | P105 | 1 | A | P104 | 1 | B |
|D3  | P104 | 1 | B | P105 | 1 | A |
|D4  | P103 | 2 | A | P106 | 0 | B |
|D5  | P102 | 2 | B | P107 | 0 | A | 2 |
|D6  | P106 | 0 | B | P111 | 3 | A | 2 |
|D7  | P107 | 0 | A | P112 | 3 | B |
|D8  | P304 | 7 | A | P304 | 7 | A |
|D9  | P303 | 7 | B | P303 | 7 | B | 1 |
|D10 | P112 | 3 | B | P103 | 2 | A | 1 |
|D11 | P109 | 1 | A | P411 | 6 | A | 1 |
|D12 | P110 | 1 | B | P410 | 6 | B |
|D13 | P111 | 3 | A | P102 | 2 | B |
|A0  | P014 |   |   | P014 |
|A1  | P000 |   |   | P000 |
|A2  | P001 |   |   | P001 |
|A3  | P002 |   |   | P002 |
|A4  | P101 | 5 | A | P101 | 5 | A |
|A5  | P100 | 5 | B | P100 | 5 | B |

普通にGPIOやSPIなどを使う分には、Arduinoライブラリが違いを隠蔽してくれるので気にする必要はなかったと思います。

ところが、モータードライバへPWM出力するためにはタイマ・チャンネルが被らないようにしなければなりません。先に書いたとおり、同じチャンネルのAとBは同時に使用できません。

Minimaの場合、D2・D3・D11・D12がチャンネル1に割り当てられています。 **Wi-Fiと同じようにD11・D12にチャンネル6を、どうしてMinimaでは使ってくれなかったのか不思議です。** 当初、基板の **〜** 記号どおりにモータードライバとつないでいたら全く動かずかなり悩みました。

ここではモータードライバ1はD9・D10・D11を使用し、モータードライバ2はD1・D5・D6を使用しました。MinimaとWi-Fiモデルのいずれも同じ構成ができるはずです。（Wi-Fiモデルは持ってないので未確認）

# コード

ドライバのインスタンスに、デジタルのソケット番号を引数にセットします。第4引数はEnableピンはどこを使ってもいいと思います。あとの初期化とloop関数での使い方に変わりはありません。

```cpp
BLDCMotor motor1{7, 26};
BLDCMotor motor2{7, 26};
BLDCDriver3PWM driver1{11, 10, 9, 8};
BLDCDriver3PWM driver2{6, 5, 1, 0};
```

# さいごに

次回はI2Cマルチプレクサをつかって磁気エンコーダを2個つなげます。

https://qiita.com/8ga3/items/61e19205d9f533660afa
