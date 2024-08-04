---
title: ジンバルモーター、SimpleFOCで動かす(9) - 磁気センサ2個接続
tags:
  - Arduino
  - SimpleFOC
  - ArduinoUnoR4
  - I2Cマルチプレクサ
  - TCA9548A
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/5f4be147a180e6b8fb06)の続きです。

Arduino Uno R4にBLDCモーターを2個接続しました。当然、磁気センサも２個になります。Arduino Uno R4とのつなぎ方とソースコードについて考えてみました。

# I2Cマルチプレクサ

前回の図を再掲します。

![BLDC_SimpleFOC_BLDCx2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/10f2b776-3909-6b4d-901a-d5930300902b.png)

AS5600とはI2C接続しています。I2Cはチップ毎にユニークな7bitのアドレスを持っていて通信相手を決めています。AS5600のアドレスは固定のため、２個つなぐとおかしなことになります。

対策方法を考えてみました。

1. マイコン側で複数のI2Cマスタを使えるの能力があるなら、それぞれに接続する。
2. I2Cマルチプレクサで切り替え
3. [I2Cアドレストランスレーター](https://akizukidenshi.com/catalog/g/g115789/)でアドレスを書き換え

1が手っ取り早いです。Arduino Uno R4はピンヘッダのA5とA6を、２つ目のI2Cとして使えます。私はこのGPIOを別の用途に使いたかったので却下しました。

あとは2と3ですが、I2Cデバイスが2個なら3が楽かなと思います。しかし私は2のマルチプレクサを使いました。単純に先に手元に届いたからです。

図の右上あたりにある **TCA9548A** が使用したICです。TCA9548A自体もI2Cのアドレスを持っていて、ICにぶら下げた磁気センサへのチャンネルを切り替えてくれます。8chも切り替えられますが、TIのラインナップを見ると2ch品・4ch品もあるみたいですね。アマゾンで5個セット￥749で入手できました。

# TCA9548A

![TCA9548A_1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b13c039f-7818-7075-9f1a-add16a5ff18b.png)

TCA9548Aモジュールにはプルアップ抵抗がついていました。よってマスター側のSCLとSDAへ直接つないで問題ないです。RESETは今回使いません。A0・A1・A2はアドレス指定になります。アドレスは表のようになります。どこにも接続しないでみたらデフォルトの70hでした。IC内部でプルダウンしているのかもしれませんがドキュメントに記載が見つからなかったので、なるべくGNDかVCCにつないだほうがよさそうです。

![TCA9548A_3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/73e134c9-a683-206d-e0f5-91ed950da6c0.png)

磁気センサはひとつめをSC0・SD0へ接続、ふたつめはSC1・SD1に接続しました。

# ソースコード

どの磁気センサと通信するかを切り替える方法は、TCA9548AにI2Cで書き込みをおこないます。下表のように8bitのそれぞれのビットがチャンネルに対応していて、ビットを立てる（1にする）と通信できます。ちなみにch0とch1を同時オンにすることも可能だそうです。書き込みをするときはいいですが、読み込みのときは最後に受信したバイトがTCA9548Aに保存されマスターがそれを読み込みますので、ちょっと使い方としては難しそうです。今回はこのような使い方はしません。

![TCA9548A_2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/0aebd4d2-431e-2c6e-5ba1-adfbe6af71e8.png)

Arduinoのコードはこんな感じで切り替えます。説明するほどのこともないですが一応。

```cpp
Wire.begin();
Wire.setClock(400000);

byte ch = 0;

Wire.beginTransmission(0x70);
Wire.write(1 << ch);
Wire.endTransmission();
```

SimpleFOCで使いやすいようにコーディングスタイルを合わせたclassを作っておきます。

```cpp:SelectableI2CTCA9548A.h
#ifndef TCA9548A_LIB_H
#define TCA9548A_LIB_H
#include "Arduino.h"
#include <Wire.h>

class SelectableI2CTCA9548A {
  public:
    static constexpr uint8_t TCA9548A_I2C_ADDR_BASE = 0x70;
    static constexpr uint8_t TCA9548A_I2C_ADDR = TCA9548A_I2C_ADDR_BASE + 0;

    SelectableI2CTCA9548A(uint8_t _tca_addr = TCA9548A_I2C_ADDR)
      : tca_addr(_tca_addr) {
    }

    void init(TwoWire* _wire = &Wire) {
      wire = _wire;
    }

    void setTcaChannel(uint8_t _ch);

  protected:
    uint8_t tca_addr;
    TwoWire* wire;
};

#endif
```

```cpp:SelectableI2CTCA9548A.cpp
#include "SelectableI2CTCA9548A.h"

void SelectableI2CTCA9548A::setTcaChannel(uint8_t _ch) {
  if (_ch > 7) return;
  wire->beginTransmission(tca_addr);
  wire->write(1 << _ch);
  wire->endTransmission();
}
```

# MagneticSensorI2C classを継承する

`MagneticSensorI2C` classを拡張し、TCA9548Aでチャンネルを切り替えて通信できるようにします。

もともと基底クラスである `Sensor` で純粋仮想関数が定義されているため拡張可能です。`MagneticSensorI2C` のinit関数にvirtualキーワードがないことに注意が必要です。親クラスにキャストすると、子クラスのオーバーライドしたメソッドはコールできなくなります。できればSimpleFOCライブラリでvirtualをつけるように変更して欲しいと思っています。

作成したクラスはこちらです。前出の `SelectableI2CTCA9548A` と `MagneticSensorI2C` を多重継承しています。`SelectableI2CTCA9548A` はメンバ変数にしてもいいので迷いました。

```cpp:MagneticSensorSelectableI2C.h
#ifndef MAGNETICSENSORSELECTABLEI2C_LIB_H
#define MAGNETICSENSORSELECTABLEI2C_LIB_H
#include "Arduino.h"
#include <Wire.h>
#include "sensors/MagneticSensorI2C.h"
#include "SelectableI2CTCA9548A.h"

class MagneticSensorSelectableI2C: public SelectableI2CTCA9548A, public MagneticSensorI2C {
  public:
    MagneticSensorSelectableI2C(uint8_t _chip_address, int _bit_resolution, uint8_t _angle_register_msb, int _msb_bits_used, uint8_t _ch = 0, uint8_t _tca_addr = TCA9548A_I2C_ADDR);

    MagneticSensorSelectableI2C(MagneticSensorI2CConfig_s config, uint8_t _ch = 0, uint8_t _tca_addr = TCA9548A_I2C_ADDR);

    void init(TwoWire* _wire = &Wire);

    float getSensorAngle() override;

  protected:
    uint8_t ch;
};

#endif
```

```cpp:MagneticSensorSelectableI2C.cpp
#include "MagneticSensorSelectableI2C.h"

MagneticSensorSelectableI2C::MagneticSensorSelectableI2C(uint8_t _chip_address, int _bit_resolution, uint8_t _angle_register_msb, int _msb_bits_used, uint8_t _ch, uint8_t _tca_addr)
  : SelectableI2CTCA9548A(_tca_addr),
  MagneticSensorI2C(_chip_address, _bit_resolution, _angle_register_msb, _msb_bits_used) {
  ch = _ch < 8 ? _ch : 0;
}

MagneticSensorSelectableI2C::MagneticSensorSelectableI2C(MagneticSensorI2CConfig_s config, uint8_t _ch, uint8_t _tca_addr)
  : SelectableI2CTCA9548A(_tca_addr), MagneticSensorI2C(config) {
  ch = _ch < 8 ? _ch : 0;
}

/** sensor initialise pins */
void MagneticSensorSelectableI2C::init(TwoWire* _wire) {
  this->SelectableI2CTCA9548A::init(_wire);
  this->MagneticSensorI2C::init(_wire);
}

// implementation of abstract functions of the Sensor class
/** get current angle (rad) */
float MagneticSensorSelectableI2C::getSensorAngle() {
  setTcaChannel(ch);
  return this->MagneticSensorI2C::getSensorAngle();
}
```

使い方のショートコードです。

```cpp
#include <Arduino.h>
#include <SimpleFOC.h>
#include "MagneticSensorSelectableI2C.h"

MagneticSensorSelectableI2C sensor1{AS5600_I2C, 0}; // channel 0
MagneticSensorSelectableI2C sensor2{AS5600_I2C, 1}; // channel 1

void setup() {
  Serial.begin(115200);
  Wire.begin();
  Wire.setClock(400000);

  sensor1.init();
  sensor2.init();
}

void loop() {
  Serial.print(sensor1.getAngle());
  Serial.print("\t");
  Serial.println(sensor2.getAngle());
  delay(100);
}
```

子クラス内でTCA9548Aの切り替え処理を隠蔽しているので、通常の `MagneticSensorI2C` と同じように扱うことができました。MotorとMotorDriverへの登録は、以前の記事で書いたモーター1個の場合を、2個目のモーターでも繰り返すだけなので割愛します。

# さいごに

![twe_bldc.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e658f27d-df2e-c562-1eb7-7f981611378c.gif)

2個のBLDCモーターを動かしている様子です。2軸カメラジンバルを作れますね。カメラジンバルは磁気センサではなくIMUを使いますけど笑

記事で書いたセンサーやモーターとは別の品を入手して実験を続けてますが、記録していくほどの話題はないようです。というわけで今回で区切りをつけます。
