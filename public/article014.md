---
title: ジンバルモーター、SimpleFOCで動かす(4) - Teleplotでグラフ表示
tags:
  - Arduino
  - PlatformIO
  - BLDC
  - SimpleFOC
  - ArduinoUnoR4
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/c0005c7f8fe284ec52a5)の続きです。

Visual Studio Codeの機能拡張、[Teleplot](https://marketplace.visualstudio.com/items?itemName=alexnesnes.teleplot)でグラフ表示をします。

Arduino IDE 2.0のプロット表示に相当します。

# インストール

VSCodeの機能拡張からTeleplotを検索することで簡単に見つけられます。ボタンを押してインストールします。

![teleplot-marketplace.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ef42404f-19e6-8ff9-806c-716034e3931b.png)

# 使い方

使い方は非常に簡単で、Arduinoのloop関数でシリアルにプロットしたい数値をprintします。

基本は下の例のように書きます。書式外の文字列はTeleplotのコンソールに表示されます。

```cpp
loop() {
  ...

  Serial.print(">sin:");
  Serial.println(sin(i));

  ...
}
```

Arduinoに実行イメージをアップロードしリブートしたら、Teleplot画面を表示します。左下のTeleplotの文字をクリックします。

![teleplot-btn.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b70e0967-fe21-5174-3ebc-993158fb112b.png)

タブが開き、初期画面が現れます。

![teleplot-start.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/db0ce4be-7796-13db-5923-01c0b7994677.png)

シリアルデバイスとボーレートを選択し、Openボタンを押すとグラフがリアルタイムに表示されます。

![teleplot-run.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e8fae1f6-9122-7cfe-9976-61abb13e53cf.png)

UDP対応しています。複数グラフを１つに重ねて表示したり、文字列を送信することができます。

画面レイアウトをjson形式でエクスポート・インポートできるので、jsonファイルを直接編集すると、線の色を変えるとか細かい指定ができました。

詳細は割愛します。シンプルなのでちょっと試しに操作してみれば、すぐに理解できると思います。

# SimpleFOCの場合のコーディング方法

SimpleFOCでは、角度や速度等の表示にmonitor機能を使ってきました。これをやめてTeleplotのフォーマットで出力してあげます。

自前で実装してもよいのですが、すでに用意されているのでそちらを使います。

これまで使っていたSimpleFOCライブラリのライブラリにはありません。SimpleFOCDriversの方に入っているので、PlatformIOのLibrariesからインストールします。

リポジトリ[Arduino-FOC-drivers](https://github.com/simplefoc/Arduino-FOC-drivers)をみると、追加のモータードライバやエンコーダが多数ありますが、それ以外の追加機能が入っていることがわかります。ここで使うのは **TeleplotTelemetry** です。

ヘッダファイルをincludeし、グローバルにインスタンスを作ります。

```cpp
#include <comms/telemetry/TeleplotTelemetry.h>

TextIO io = TextIO(Serial);
TeleplotTelemetry telemetry = TeleplotTelemetry();
```

setup関数では、表示したい項目を登録し初期化します。

```cpp
  telemetry.addMotor(&motor);
  uint8_t reg[] = { REG_TARGET, REG_ANGLE, REG_VELOCITY };
  telemetry.setTelemetryRegisters(3, reg);
  telemetry.init(io);
  telemetry.downsample = 100;
```

ちなみにsetTelemetryRegisters関数の書き方で、下記の書き方だとエラーになりました。Arduino Uno R4に乗っているRenesas RA4M1のtoolchainがGCC 7.2.1でした。この書き方ができる前のGCCのバージョンのようです。ESP32やNucleoとかなら大丈夫なのかもしれません。

```cpp
  telemetry.setTelemetryRegisters(3, { REG_TARGET, REG_ANGLE, REG_VELOCITY });
```

loop関数では、run関数をコールするだけでシリアルにプリントされます。

```cpp
void loop()
{
  motor.loopFOC();
  motor.move();
  // motor.monitor(); // telemetryと同時に使わないので消します。
  telemetry.run();
  command.run();
}
```

ビルド、アップロードしたら準備完了です。

# Teleplotでモーターの状態表示

Teleplot画面を開き、ポートを設定しOpenボタンを押せばグラフが表示されると思います。あとは使いやすいように画面レイアウトをカスタマイズしましょう。コンソールからCommanderへ司令することもできます。

![teleplot-run2.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e3d002f9-cf42-8417-b171-c18c5f1f71c4.gif)


# おわりに

Teleplotを使えばPIDの調整がしやすくなったかと思います。

次回は、今回使用した磁気エンコーダについて少し書いてみます。
