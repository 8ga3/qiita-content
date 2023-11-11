---
title: M5StackのUnit EncoderをM5Unifiedと共に使うClass
tags:
  - Arduino
  - PlatformIO
  - M5stack
  - M5Unified
  - UnitEncoder
private: false
updated_at: '2023-11-11T10:27:56+09:00'
id: 50fd5c0691ea8f025d9d
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

前回、M5StackのUnit Encoderのファームウェアをアップデートしました。追加された機能が使えて、いつも利用しているM5Unifiedと連携できるクラスがほしかったので作りました。M5Unifiedの書き方に準じつつ、Unit EncoderのExampleのメンバ関数とだいたい同じ感じになるようにしています。

https://qiita.com/8ga3/items/ed60578a1d55e32bfc2b

# ライブラリ共有

GitHubにリポジトリを作りました。`lib/UnitEncoder_Class/UnitEncoder_Class.cpp`が該当します。platformioのプロジェクトの形になってます。

https://github.com/8ga3/UnitEncoder_Class_M5Unified

# 初期化

setup関数でbegin()をコールしてください。PORT.AのI2Cに繋がっているものとして初期化されます。

```cpp:main.cpp
#include <M5Unified.h>
#include <UnitEncoder_Class.hpp>

m5::UnitEncoder_Class UnitEncoder;

void setup()
{
  auto cfg = M5.config();
  M5.begin(cfg);
  UnitEncoder.begin();
}
```

# メンバ関数

## setEncoderMode (V1.1)

```cpp
void setEncoderMode(encoder_mode_t mode);
```

ロータリーエンコーダーのモードを設定します。

パラメータ
* *mode* :

| name | 説明 |
|:-----|:-----|
|m5::UnitEncoder_Class::encoder_mode_t::mode_pulse | 時計回りはカウントアップ、反時計回りはカウントダウンします。 |
|m5::UnitEncoder_Class::encoder_mode_t::mode_ab | 絶対角度 |

## getEncoderValue

```cpp
signed short int getEncoderValue();
```

ロータリーエンコーダーの値を取得します。setEncoderMode()の設定によって取得できる値は変わります。

## setEncoderValue (V1.1)

```cpp
bool setEncoderValue(signed short int value);
```

ロータリーエンコーダーの値を設定します。起動直後は0が設定されています。

パラメータ
* *value* : -32768 ~ 32767

## getButtonStatus

```cpp
bool getButtonStatus();
```

ボタンが押されているかどうかを取得します。

## setLEDColor

```cpp
void setLEDColor(uint8_t index, uint32_t color);
```

LED色を設定します。

パラメータ
* *index* : LED番号
  * 0: すべてのLED
  * 1: LED1
  * 2: LED2
  * その他: 未定義
* *color* : 0x00RRGGBBの順のフルカラー指定

## resetCounter (V1.1)

```cpp
bool resetCounter();
```

リセットします。

## おわりに

今後も手持ちのM5Stackモジュールを、M5Unified対応にして使っていきたいと思います。少ししか持っていませんけど。
