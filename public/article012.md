---
title: ジンバルモーター、SimpleFOCで動かす(2)
tags:
  - Arduino
  - PlatformIO
  - BLDC
  - SimpleFOC
  - ArduinoUnoR4
private: false
updated_at: '2024-07-04T01:04:45+09:00'
id: 9448e5cb4ce6a69effb0
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/e4d768db56cdb59da31a)の続きです。

https://qiita.com/8ga3/items/e4d768db56cdb59da31a

# ソースコード

ほとんど[公式の例](https://docs.simplefoc.com/code)のままです。違いのある部分を説明します。

```cpp
#include <Arduino.h>
#include <SimpleFOC.h>

BLDCMotor motor = BLDCMotor(7, 26);
BLDCDriver3PWM driver = BLDCDriver3PWM(11, 10, 9, 8);
MagneticSensorI2C sensor = MagneticSensorI2C(AS5600_I2C);

Commander command = Commander(Serial);
void onTarget(char* cmd){ command.motion(&motor, cmd); }

void setup()
{
  pinMode(12, OUTPUT); // GND

  Serial.begin(115200);
  SimpleFOCDebug::enable();

  // PC側のシリアルモニターと接続するまでの待ち時間
  _delay(1000);
  SIMPLEFOC_DEBUG("Setup");

  sensor.init();
  motor.linkSensor(&sensor);

  driver.voltage_power_supply = 12;
  driver.init();
  motor.linkDriver(&driver);

  motor.controller = MotionControlType::angle;

  motor.PID_velocity.P = 0.006f;
  motor.PID_velocity.I = 0.020f;
  motor.PID_velocity.D = 0.00001f;

  motor.voltage_limit = 6.0f;

  motor.LPF_velocity.Tf = 0.02f;

  motor.P_angle.P = 2.0f;
  motor.velocity_limit = 20.0f;

  // Monitor
  motor.useMonitoring(Serial);
  motor.monitor_downsample = 100; // default DEF_MON_DOWNSMAPLE(100)
  motor.monitor_decimals = 4; // default 4
  motor.monitor_variables = _MON_TARGET | _MON_VEL | _MON_ANGLE; // monitor target velocity and angle

  motor.init();
  motor.initFOC();

  command.add('T', onTarget, "motion control");

  SimpleFOCDebug::println("Motor ready.");
  SimpleFOCDebug::println("Set the target angle using serial terminal:");
  _delay(1000);
}

void loop()
{
  motor.loopFOC();
  motor.move();
  motor.monitor();
  command.run();
}
```

# モーター・ドライバー・磁気センサー設定

```cpp
BLDCMotor motor = BLDCMotor(7, 26);
```

7は極(ポール)のペア数となっています。極は磁石のことです。このモーターはアウターローターなので、外側に磁石がついています。数えると14個のため半分の7としています。このくらいのサイズだと12とか14が多いのではないでしょうか。
ちなみにステーター(固定子)は12スロットでした。コイルのターン数は80です。

![IMG_6692.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/2013f071-2170-c8cb-9193-c51205f792c8.jpeg)

26は抵抗値です。モーターのデーターシートがないので測定しました。ちゃんと結線を確認はしていないのですが、おそらくデルタ結線であろうということで計算しています。

最初に3相のうち2本の抵抗値をテスターで測ります。2本の組み合わせを3パターン計測し、平均すると17.5Ωでした。この計測値に1.5倍すると相単位の抵抗値です。

電流センサーをつけていないので、抵抗値から推定するのに使うのでしょうか。おおよそでいいと思います。SimpleFOCMiniは10Ω以上のモーターに対応しているそうです。

Y結線だったら間違いになりますが、動いたからヨシとしました。

![3phase.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/34e50cb4-4607-d454-cd96-1245c11b2acc.png)

```math
\frac{1}{\frac{1}{R+R} + \frac{1}{R}}=\frac{2}{3}R
```

# Arduino Uno R4 のハードウェア機能有効化

実はこれに悩まされました。マイコン毎に[コード](https://github.com/simplefoc/Arduino-FOC/tree/master/src/drivers/hardware_specific)があります。私はArduino Uno R4を使ったので、Renesas用コードを使ってほしかったわけです。Arduino IDEやPlatformIOなら自動的に切り替わるようになっていました。

ビルド中にメッセージが出力されているかで確認できます。

```
.pio/libdeps/uno_r4_minima/Simple FOC/src/drivers/hardware_specific/renesas/renesas.cpp:8:68: note: #pragma message: SimpleFOC: compiling for Arduino/Renesas (UNO R4)
 #pragma message("SimpleFOC: compiling for Arduino/Renesas (UNO R4)")
```

ところがビルドして書き込んだところ、Renesas用コードが実行されている感じがしません。

なぜそう思うかというと、モーターからビーという音が出続けています。PWMの周波数が低いためです。Renesasがこんなに低いわけ無いです。デフォルトは[24000Hz](https://github.com/simplefoc/Arduino-FOC/blob/6fafa35b73e9c5ba6f5d9259640a5ed328dc2750/src/drivers/hardware_specific/renesas/renesas.h#L13C39-L13C44)のはずです。

ひとまず、`platformio.ini`にビルド・フラグ`SIMPLEFOC_RENESAS_DEBUG`を追加して様子を見てみます。

```ini:platformio.ini
build_flags = 
	-D SIMPLEFOC_RENESAS_DEBUG
```

本来なら以下のような文字列がシリアルモニターで確認できるはずです。しかし表示されません。

```
---PWM Config---
DRV: pwm pin: 11
DRV: pwm channel: 1
DRV: pwm A/B: A
DRV: pwm freq: 24000
DRV: pwm range: 1000
DRV: pwm clkdiv: 0
---PWM Config---
DRV: pwm pin: 10
DRV: pwm channel: 3
DRV: pwm A/B: B
DRV: pwm freq: 24000
DRV: pwm range: 1000
DRV: pwm clkdiv: 0
---PWM Config---
DRV: pwm pin: 9
DRV: pwm channel: 7
DRV: pwm A/B: B
DRV: pwm freq: 24000
DRV: pwm range: 1000
DRV: pwm clkdiv: 0
DRV: starting timer: 1
DRV: starting timer: 3
DRV: starting timer: 7
DRV: timers started
```

そして原因ですが、[generic_mcu.cpp](https://github.com/simplefoc/Arduino-FOC/blob/master/src/drivers/hardware_specific/generic_mcu.cpp)のせいでした。[renesas.cpp](https://github.com/simplefoc/Arduino-FOC/blob/master/src/drivers/hardware_specific/renesas/renesas.cpp)と比べてみると関数名が同一です。これではリンクしたときに、どちらの関数が使われるかは運になるのではないでしょうか。

とりあえず私は`generic_mcu.cpp`の中身をまるごと`#if 0 〜 #endif`で囲って、実質削除してしまいました。

ようやく音がなく、ガタガタ振動することもなく動くようになりました。発熱もなくなったので安心して触れます。

# シリアルモニターへデバッグ出力

ディレイを入れないとパソコン側のシリアルモニターの接続が間に合わないので、適当な時間を待たせています。必要なくなったら消してOKです。

```cpp
  Serial.begin(115200);
  SimpleFOCDebug::enable();

  // PC側のシリアルモニターと接続するまでの待ち時間
  _delay(1000);
```

`monitor_downsample`にすると100回に一度だけ数値が出るようになります。

`monitor_decimals`は小数点以下の桁数です。

`monitor_variables`でシリアルモニターに表示したい項目を設定します。

```cpp
  // Monitor
  motor.useMonitoring(Serial);
  motor.monitor_downsample = 100; // default DEF_MON_DOWNSMAPLE(100)
  motor.monitor_decimals = 4; // default 4
  motor.monitor_variables = _MON_TARGET | _MON_VEL | _MON_ANGLE; // monitor target velocity and angle
```

指定できる項目は[FOCMotor.h](https://github.com/simplefoc/Arduino-FOC/blob/master/src/common/base_classes/FOCMotor.h)に定義されています。1行にタブ区切りで数値が出力されます。

```
// monitoring bitmap
#define _MON_TARGET 0b1000000 // monitor target value
#define _MON_VOLT_Q 0b0100000 // monitor voltage q value
#define _MON_VOLT_D 0b0010000 // monitor voltage d value
#define _MON_CURR_Q 0b0001000 // monitor current q value - if measured
#define _MON_CURR_D 0b0000100 // monitor current d value - if measured
#define _MON_VEL    0b0000010 // monitor velocity value
#define _MON_ANGLE  0b0000001 // monitor angle value
```

# 動作動画

指でローターを回して離すと、元の角度へ戻っていくように動きました。

![bldc_angle.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3d600a15-401d-2b7b-29e0-1b26052fd6b9.gif)

# おわりに

アングルモードで動くようになりました。

次回、Commanderを使ってみます。

https://qiita.com/8ga3/items/c0005c7f8fe284ec52a5
