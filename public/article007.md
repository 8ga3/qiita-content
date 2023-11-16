---
title: M5StackのAtomMotionをM5Unifiedと共に使うClass
tags:
  - Arduino
  - PlatformIO
  - M5stack
  - M5Unified
  - AtomMotion
private: false
updated_at: '2023-11-16T10:42:47+09:00'
id: 0bd721cfa1a2966a432c
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

[M5StackのAtomMotion](https://shop.m5stack.com/products/atom-motion-kit-with-motor-and-servo-driver-stm32f0)を、いつも利用しているM5Unifiedと連携して使いやすくしたライブラリを作りました。[Unit Encoder用のClassを作った話](https://qiita.com/8ga3/items/50fd5c0691ea8f025d9d)と動機は同じです。

メンバ関数はM5Stack公式のスケッチ例にあるAtomMotion.(cpp|h)を継承しています。

https://qiita.com/8ga3/items/50fd5c0691ea8f025d9d

# ライブラリ共有

GitHubにリポジトリを作りました。`lib/AtomMotion_Class/AtomMotion_Class.cpp`が該当します。platformioのプロジェクトの形になってます。

https://github.com/8ga3/AtomMotion_Class_M5Unified

# 初期化

特に宣言はありません。M5Stack系・M5Stick系はサイド、M5Atom系は底のピンのI2Cにつながるものとして初期化されます。M5Unifiedによって自動判別で適切に設定されます。
変更する場合は、コンストラクタの引数で指定できます。

```cpp:main.cpp
#include <M5Unified.h>
#include <AtomMotion_Class.hpp>

m5::AtomMotion Motion;

void setup() {
  auto cfg = M5.config();
  M5.begin(cfg);

  Motion.SetServoAngle(1, 90);
}
```

# メンバ関数

## コンストラクタ

```cpp
  AtomMotion(std::uint8_t i2c_addr = DEFAULT_ADDRESS, std::uint32_t freq = 400000, I2C_Class* i2c = &In_I2C);
```

**パラメータ**
* *i2c_addr* : アドレス値。AtomMotionは0x38です。
* *freq* : クロック数
* *i2c* : I2C_ClassでSCL/SDAの番号を設定しています。

## SetServoAngle

```cpp
  std::uint8_t SetServoAngle(std::uint8_t Servo_CH, std::uint8_t angle);
```

サーボモーターを指定した角度にします。[Servo Kit 180‘](https://shop.m5stack.com/products/servo-kit-180) を使用することが前提となっていると思います。angleの値とサーボモーターの角は機種ごとに確認したほうが良いです。

**パラメータ**
* *Servo_CH* : サーボ番号(1~4)
* *angle* : 角度(0~180)

**戻り値**
* 0 : 成功
* 1 : 引数が範囲外

## SetServoPulse

```cpp
  std::uint8_t SetServoPulse(std::uint8_t Servo_CH, std::uint16_t width);
```

サーボモーターのパルス幅を設定します。1500が中央値です。360度連続回転サーボを使う場合、1500で停止するようにプラスドライバーを使ってモーター本体を調整しましょう。

**パラメータ**
* *Servo_CH* : サーボ番号(1~4)
* *width* : パルス幅(500~2500)

**戻り値**
* 0 : 成功
* 1 : 引数が範囲外

## SetMotorSpeed

```cpp
  std::uint8_t SetMotorSpeed(std::uint8_t Motor_CH, std::int8_t speed);
```

Atom Motionの２ポートあるブラシモーターの速度を設定します。０で停止、プラス・マイナスで回転方向が変わります。

**パラメータ**
* *Servo_CH* : モーター番号(1~2)
* *speed* : -127~127

**戻り値**
* 0 : 成功
* 1 : 引数が範囲外

## ReadServoAngle

```cpp
  std::uint8_t ReadServoAngle(std::uint8_t Servo_CH);
```

サーボモーターの角度を取得します。

**パラメータ**
* *Servo_CH* : サーボ番号(1~4)

**戻り値**
* 角度

## ReadServoPulse

```cpp
  std::uint16_t ReadServoPulse(std::uint8_t Servo_CH);
```

サーボモーターのパルス幅を取得します。

**パラメータ**
* *Servo_CH* : サーボ番号(1~4)

**戻り値**
* パルス幅

## ReadMotorSpeed

```cpp
  std::int8_t ReadMotorSpeed(std::uint8_t Motor_CH);
```

ブラシモーターの速度を取得します。

**パラメータ**
* *Servo_CH* : ブラシモーター番号(1~2)

**戻り値**
* 速度
