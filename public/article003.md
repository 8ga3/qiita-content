---
title: micro-ROSをM5 Atom Matrixへ書き込み
tags:
  - ATOM
  - PlatformIO
  - ROS2
  - M5stack
  - micro-ROS
private: false
updated_at: '2023-11-09T19:56:52+09:00'
id: 8535699b8a7141568263
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

M5 Atom Matrixにmicro-ROSをフラッシュします。さきに少し前に他のブログで紹介されている方法を試しましたが、ESP32がリブートを繰り返してしまってうまくいきませんでした。micro-ROSは開発の進みが早いようで、私の手順のメモを残すことにしました。

2023年11月現在の方法ですので、これもいつまで有効かわかりません。

# 使用環境とソフトウェアのバージョン

| name              | version       |
|:------------------|:--------------|
| MBP               | Intel Core i7 |
| macOS             | 13.6.1        |
| PlatformIO Core   | 6.1.11        |
| PlatformIO Home   | 3.4.4         |
| ROS2              | Iron Irwini   |
| Ubuntu            | 22.04         |
| Parallels Desktop | 19.1.0        |

# PlatformIOでプロジェクト作成

Visual Studio CodeにPlatformIOが入っているものとします。

基本的に[公式サイト](https://github.com/micro-ROS/micro_ros_platformio)に手順が書いてあります。このとおり実施すれば動作します。

:::note info
Arduino IDEをご使用の場合は、[こちら](https://github.com/micro-ROS/micro_ros_arduino)を参照してみてください。
:::

binutilsを使うのでまだの場合はインストールします。

```shell
brew install binutils
```

しかしインストールしただけですとビルドに失敗します。M1Mac以降、インストール先が`/usr/local/`から`/opt/homebrew/`に変わりました。PlatformIOも新しい方が標準になったようです。変更方法がわからなかったのでシンボリックリンクを張って凌ぐことにしました。

```shell
cd /opt/homebrew/opt
sudo ln -s ../../../usr/local/opt/binutils binutils
```

```platform.ini```を以下のように編集します。`lib_deps`に`Wire`がないとビルドエラーになるので省略しないようにします。

:::note info
M5 Atom S3の設定を追加しました。
M5Unifiedを使うように変えました。Wireは必要ありません。
:::

```ini:platform.ini
[env:m5stack-atom]
platform = espressif32
board = m5stack-atom
framework = arduino
lib_deps = 
    fastled/FastLED@^3.6.0
  	m5stack/M5Unified@^0.1.10
    https://github.com/micro-ROS/micro_ros_platformio
board_microros_distro = iron
board_microros_transport = serial

[env:m5stack-atoms3]
platform = espressif32
board = m5stack-atoms3
framework = arduino
lib_deps = 
	fastled/FastLED@^3.6.0
	m5stack/M5Unified@^0.1.10
	https://github.com/micro-ROS/micro_ros_platformio
build_flags =
	'-D ARDUINO_USB_MODE=1'
	'-D ARDUINO_USB_CDC_ON_BOOT=1'
board_microros_distro = iron
board_microros_transport = serial
```

`board_microros_distro`を記述しない場合はデフォルトになります。
* `humble`
* `iron` (default value)
* `rolling`

`board_microros_transport`は、Atomからの転送方法です。ここではUSBシリアルを使ってます。この指定に合わせてソースコードを記載します。書き方は[READEME.md](https://github.com/micro-ROS/micro_ros_platformio#transport-configuration)を参照してください。

* `serial` (default value)
* `wifi`
* `wifi_nina`
* `native_ethernet`
* `custom`

# コーディング

[example](https://github.com/micro-ROS/micro_ros_platformio/blob/main/examples/micro-ros_publisher/src/Main.cpp)をコピペします。今回はAtom Matrixを使用しています。IMUにMPU6886が搭載されているので、3軸重力加速度計と3軸ジャイロスコープをpublishするように変更してみます。

```cpp:main.cpp
#include <Arduino.h>
#include <M5Unified.h>
#include <micro_ros_platformio.h>

#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>

#include <sensor_msgs/msg/imu.h>

#if !defined(MICRO_ROS_TRANSPORT_ARDUINO_SERIAL)
#error This example is only avaliable for Arduino framework with serial transport.
#endif

rcl_publisher_t publisher;
sensor_msgs__msg__Imu msg;

rclc_executor_t executor;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_node_t node;
rcl_timer_t timer;

#define RCCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){error_loop();}}
#define RCSOFTCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){}}

// Error handle loop
void error_loop() {
  while(1) {
    delay(100);
  }
}

void timer_callback(rcl_timer_t * timer, int64_t last_call_time) {
  RCLC_UNUSED(last_call_time);

  if (timer != NULL) {
    float linear_acceleration_x = 0.0;
    float linear_acceleration_y = 0.0;
    float linear_acceleration_z = 0.0;
    float angular_velocity_x = 0.0;
    float angular_velocity_y = 0.0;
    float angular_velocity_z = 0.0;

    M5.IMU.getGyroData(&angular_velocity_x, &angular_velocity_y, &angular_velocity_z);
    M5.IMU.getAccelData(&linear_acceleration_x, &linear_acceleration_y, &linear_acceleration_z);

    msg.linear_acceleration.x = linear_acceleration_x;
    msg.linear_acceleration.y = linear_acceleration_y;
    msg.linear_acceleration.z = linear_acceleration_z;
    msg.angular_velocity.x = angular_velocity_x;
    msg.angular_velocity.y = angular_velocity_y;
    msg.angular_velocity.z = angular_velocity_z;

    RCSOFTCHECK(rcl_publish(&publisher, &msg, NULL));
  }
}

void setup() {
  auto cfg = M5.config();
  M5.begin(cfg);

  // Configure serial transport
  Serial.begin(115200);
  set_microros_serial_transports(Serial);
  delay(2000);

  allocator = rcl_get_default_allocator();

  //create init_options
  RCCHECK(rclc_support_init(&support, 0, NULL, &allocator));

  // create node
  RCCHECK(rclc_node_init_default(&node, "micro_ros_platformio_node", "", &support));

  // create publisher
  RCCHECK(rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(sensor_msgs, msg, Imu),
    "pub_imu"));

  // create timer,
  const unsigned int timer_timeout = 1000;
  RCCHECK(rclc_timer_init_default(
    &timer,
    &support,
    RCL_MS_TO_NS(timer_timeout),
    timer_callback));

  // create executor
  RCCHECK(rclc_executor_init(&executor, &support.context, 1, &allocator));
  RCCHECK(rclc_executor_add_timer(&executor, &timer));
}

void loop() {
  delay(100);
  RCSOFTCHECK(rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100)));
}
```

# フラッシュ

PlatformIOのUploadボタンを押してフラッシュします。vscodeの左下あたりにボタンが並んでます。

# micro_ROS_Agent

macOSで動かそうとしましたが失敗しました。そもそもROS2はmacOS未対応です。そこでdockerを使って簡単にagentを動かす方法が提供されています。しかし、macOSだとUSBシリアルをdocker側にマウントができませんでした。qemuの設定か何かあるのかもしれませんが、素直にUbuntuで動かすことにします。私はUbuntuマシンを持ってないので、Parallels DesktopにUbuntu 22.04をインストールしました。

## ROS2 Iron Irwini インストール

[公式ドキュメント](https://docs.ros.org/en/iron/Installation/Ubuntu-Install-Debians.html)にそって実施すれば問題ありません。私はaptでros-iron-desktopを入れています。

## micro_ROS ビルド

[micro_ros_setupのビルド方法](https://github.com/micro-ROS/micro_ros_setup#building)を実施します。

```shell
source /opt/ros/iron/setup.bash
mkdir uros_ws && cd uros_ws
git clone -b iron https://github.com/micro-ROS/micro_ros_setup.git src/micro_ros_setup
rosdep update && rosdep install --from-paths src --ignore-src -y
colcon build
source install/local_setup.bash
```

## USBシリアルの準備

M5 Atomをつなぎ、Parallels Desktopの設定でゲストOS側が使えるように変更します。`/dev/ttyUSB0`があればOKです。

```shell
$ ls -l /dev/ttyUSB0 
crw-rw---- 1 root dialout 188, 0 Nov  6 11:24 /dev/ttyUSB0
```

lsで表示するとdialoutグループに入ってないと使用できません。ユーザーをグループに入れてあげます。

```shell
$ sudo adduser [username] dialout
```

一度リブートするかログインし直します。
idコマンドでdialoutが表示されていることを確認します。

```shell
$ id
uid=1000(username) gid=1000(username) groups=1000(username),4(adm),20(dialout),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),134(lxd),135(sambashare)
```

## micro_ROS_Agent ビルド

[micro_ROS_Agentのビルド方法](https://github.com/micro-ROS/micro_ros_setup#building-micro-ros-agent)を実施します。

```shell
ros2 run micro_ros_setup create_agent_ws.sh
ros2 run micro_ros_setup build_agent.sh
source install/local_setup.sh
```

## 実行

ビルドで使ったターミナルでagentを実行します。

```shell
$ ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -b 115200 -v6
```

別のターミナルを動かし、topicがあることを確認します。`/pub_imu`があれば成功です。

```shell
$ source /opt/ros/iron/setup.bash
$ ros2 topic list
/parameter_events
/pub_imu
/rosout
```

送信されてくるデータの中身を表示してみましょう。

```shell
$ ros2 topic echo /pub_imu
```

左側ターミナルがmicro_ROS_Agent、右側ターミナルがtopic表示です。1秒毎に送られてくることがわかります。

![run_micro-ros-agent.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f79e6194-708d-32e9-aa64-f6ca291b0f19.png)


## dockerで動かす場合

Ubuntu上でdockerを[公式ドキュメント](https://docs.docker.com/engine/install/ubuntu/)に沿ってインストールします。準備ができたら実行します。

```shell
$ sudo docker run -it --rm -v /dev:/dev -v /dev/shm:/dev/shm --privileged --net=host microros/micro-ros-agent:iron serial --dev /dev/ttyUSB0 -v6
```

ROS2で開発を進めるのであれば、dockerを使わずROS2環境を作ってしまったほうが早いような気がします😜

# トラブル・シューティング

M5 Atomをフラッシュした直後や、Parallels Desktopに接続したとき、シリアルにデータが送信されませんでした。M5 Atomのリセットボタンを押してあげてください。Agentとの接続を切ったときもリセットが必要な感じです。必ずリセットが必要なのか、まだよくわかってません。
