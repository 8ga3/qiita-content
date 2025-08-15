---
title: pioarduino IDE (PlatformIO)でpymcuprogを使ってATtiny202に書き込む方法
tags:
  - Arduino
  - PlatformIO
  - ATtiny202
  - tinyAVR
  - pymcuprog
private: false
updated_at: '2025-08-15T14:22:46+09:00'
id: 676b94cfa88baa1b6b3d
organization_url_name: access
slide: false
ignorePublish: false
---
# 概要

Visual Studio CodeでPlatformIOを使用して、`pymcuprog`を介してATtiny202に書き込む手順を説明します。

ATtiny202以外のtinyAVR® 0系統は同様の手順で書き込みが可能です。おそらく1系統と2系統も可能だと思います。

:::note info
UPDI (Unified Program and Debug Interface) は、MicrochipのtinyAVR®マイコン向けに設計されたプログラミングおよびデバッグインターフェースです。
[UPDI Friend](https://ssci.to/9535)などを購入するか、USBシリアル変換モジュールを使って自作することも可能です。
:::

ネット上には同様の記事がありますが、pythonの仮想環境を使っているところが相違点です。

# 環境

- ATtiny202
  - [秋月電子通商 ATtiny202 DIP化キット](https://akizukidenshi.com/catalog/g/g118304/)
- UART接続
  - [秋月電子通商 CH340E USBシリアル変換モジュール Type-C](https://akizukidenshi.com/catalog/g/g114745/)
- 開発環境
  - Visual Studio Code
  - pioarduino (PlatformIO)
  - [pymcuprog](https://github.com/microchip-pic-avr-tools/pymcuprog)
  - Apple M3 Pro
  - macOS Sequoia 15.6
  - python 3.13.6

# プロジェクト作成

Copilotを使いたいので、Visual Studio Code上で開発することを前提としています。拡張機能でpioarduinoをインストールしておきます。（PlatformIOでもよいですが、ESP32方面でいろいろ困るので変えました。）

Arduino IDEは2.x系では未対応なので1.x系を使用することになります。私は古いバージョンを使い続けることに抵抗を感じます。Copilotも使いたいのでVScodeを選びました。

プロジェクトを作成します。

![article022_img1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a0aabace-7f90-4530-b64d-6ed3cd2c9155.png)

# UPDI書き込み作成

[秋月電子通商 ATtiny202 DIP化キット](https://akizukidenshi.com/catalog/g/g118304/)に合わせて、6ピンで繋げられるように作成しました。

- 部品
  - [ショットキーバリアダイオード 30V200mA BAT43](https://akizukidenshi.com/catalog/g/g113907/)
  - [カーボン抵抗(炭素皮膜抵抗) 1/6W470Ω](https://akizukidenshi.com/catalog/g/g116471/)

動作確認用にLチカしたいので、適当なLEDと抵抗も追加しました。

安定しない場合は、電源に10μF〜程度のコンデンサを追加するとよいでしょう。

![article022_img2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ced8d122-2de2-49d7-b052-1b5acf4910e2.png)

# pymcuprog インストール

Pythonの仮想環境にvenvを使って`pymcuprog`をインストールします。

```bash
$ python3.13 -m venv .venv
$ source .venv/bin/activate
$ pip install pymcuprog
```

ATTiny202をUPDIで接続します。`pymcuprog`を使ってpingコマンドで接続確認を行います。

```bash
$ pymcuprog ping -t uart -u /dev/tty.usbserial-110 -d attiny202 -c 57600
Connecting to SerialUPDI
Pinging device...
Ping response: 1E9123
Done.
```

# platformio.ini 編集

`platformio.ini`を編集して、`pymcuprog`を使うように設定します。[こちら](https://docs.platformio.org/en/latest/platforms/atmelmegaavr.html)に書き方のサンプルがあります。このまですとpython仮想環境を使わないので、以下のように書き換えます。

`upload_port`はマシン環境に合わせます。

`--verify`と`--erase`はオプションです。フラッシュメモリの状態によってはベリファイが失敗することがあります。対策として、フラッシュメモリを消去してから書き込みを行ってます。

`upload_command`では、`source ./.venv/bin/activate`で仮想環境を有効化してから、`pymcuprog`を実行します。

```ini:platformio.ini
[env:ATtiny202]
platform = atmelmegaavr
board = ATtiny202
framework = arduino
upload_protocol = custom
upload_port = /dev/tty.usbserial-110
upload_speed = 57600
upload_flags =
    --tool
    uart
    --device
    attiny202
    --uart
    $UPLOAD_PORT
    --clk
    $UPLOAD_SPEED
    --verify
    --erase
upload_command = source ./.venv/bin/activate && pymcuprog write $UPLOAD_FLAGS --filename $SOURCE
```

# Lチカ

テストプログラムです。`delay`の代わりに`_delay_ms`を使うと、ビルド後のコードサイズが小さくなります。ATtiny202はFlashが2KBしかないので、できるだけ小さくしたいところです。

```cpp
#include <Arduino.h>
#include <util/delay.h>

const uint8_t check_led = PIN_PA3;

void setup() {
  pinMode(check_led, OUTPUT);
  digitalWrite(check_led, LOW);
}

void loop() {
  digitalWrite(check_led, LOW);
  _delay_ms(1000);
  digitalWrite(check_led, HIGH);
  _delay_ms(1000);
}
```

# まとめ

書き込みしLチカしたら成功です。お疲れ様でした。
