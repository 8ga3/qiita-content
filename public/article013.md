---
title: ジンバルモーター、SimpleFOCで動かす(3)
tags:
  - Arduino
  - PlatformIO
  - BLDC
  - SimpleFOC
  - ArduinoUnoR4
private: false
updated_at: '2024-07-04T01:01:21+09:00'
id: c0005c7f8fe284ec52a5
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/9448e5cb4ce6a69effb0)の続きです。

Commanderについての紹介をします。

https://qiita.com/8ga3/items/9448e5cb4ce6a69effb0

# Commanderとは

シリアルモニターを通じて、モーターを操作したりパラメータを変更する機能となります。

公式のドキュメントは[こちら](https://docs.simplefoc.com/commander_interface)になります。

https://docs.simplefoc.com/commander_interface

# コード説明

前記事の[ソースコード](https://qiita.com/8ga3/items/9448e5cb4ce6a69effb0#%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%89)から抜粋します。いちばん基本的なところを説明しようと思います。

Commanderのインスタンスを作成します。

onTarget関数はコールバック用です。

とりあえず、お決まりのパターンだと思ってコーディングしましょう。

```cpp
Commander command = Commander(Serial);
void onTarget(char* cmd){ command.motion(&motor, cmd); }
```

setup関数でmotorオブジェクトの設定と初期化のあと、Commanderのメンバ関数addを使って登録します。シリアルモニターを通じてキーボード入力で操作できるようになります。

第1引数はコマンドの最初に入力する文字です。なんでもいいです。大文字・小文字は区別されます。モーターが複数あるときは、違う文字を使ってモーターの数だけaddします。onTarget関数側でswitchなどを使って個別に制御できます。もちろんまとめて動かすことだってできますね。

第2引数はコールバック関数です。

第3引数はラベルです。シリアルモニターで受信した文字列にラベルが入るのでわかりやすくなります。必要なければNULLにします。

```cpp
void setup()
{

  ...

  command.add('T', onTarget, "motion control");

  ...

}
```

loop関数ではメンバ関数runをコールするだけです。

```cpp
void loop()
{
  motor.loopFOC();
  motor.move();
  motor.monitor();
  command.run();
}
```

# 使い方

詳しい使い方は公式ドキュメントを見てください。

ここではCommander::motion()の使い方を紹介します。

まず前提として、Visual Studio Code拡張機能の[Serial Monitor](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-serial-monitor)を使っています。インストールされているものとします。自動再接続が便利で、PlatformIOでArduinoへUploadのときに接続が一度切れても、マイコン側がリブートしたあとに何もしなくてもポートが開きます。再接続できるかポーリングしているようで、Arduino側はシリアルポートを開いてから文字出力するまえに1秒くらいディレイを入れてあげると、文字が欠けることがなくなると思います。

シリアルモニターのタブはこんな感じです。一番下の行に送信したいコマンドを入れます。

![serialmon.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/09470687-c50b-d8da-6ea3-41733c7377d3.png)

前回のソースコードは、初期状態で角度制御モードにしています。それではコマンドを入れてモーターを動かしてみましょう。

最初に`T`とし2文字目から角度を入れます。ラジアンですので、例のように3.14なら180度回転します。マイナスをつければ反対に回ります。どちらに回るかはモーターの3ピンコネクタの刺した向きによるので、自分で決めておいたほうが良いと思います。わたしは時計回り(CW)を＋、反時計回り(CCW)をーにしています。

一周の$2π=6.28$を超えた場合は、超えた分だけ回ります。T100だと16回転くらいします。

```shell
T3.14
```

コマンドを受け付けると、以下のようなメッセージを受信します。

```shell
Target: 3.14
```

モード変更するためにはコマンド`C`を使います。`TC1`を送信すると速度指定モードになります。数値を入れないと現在のモードがなにか教えてくれます。

```shell
TC1
```

コマンドは6個あります。Dだけサブコマンドとなっていて、ダウンサンプリング数を変更できます。初期値は0でダウンサンプリングなしです。

3と4はオープンループです。角度センサーを付ける前に、とりあえずモーターだけ確認するとか使ってもいいと思います。

* D - downsample motion loop
* 0 - トルク
* 1 - 速度
* 2 - 角度
* 3 - 速度（オープンループ）
* 4 - 角度（オープンループ）

モードを変更すると、以降のコマンドで`T`に続く数値の意味は変わります。そのままですが、速度とトルクの指定値になります。制御モードによってはスペース区切りで数値を複数渡せます。

```shell
# トルク指定
TC0
T2.0

# 速度指定（オープンループも同一）
TC1
# オプションの2番目の数値はトルク制限を変更、省略可能
T3.0 2.0

# 角度指定（オープンループも同一）
TC2
# オプションの2番目の数値は速度制限を変更、省略可能
# オプションの3番目の数値はトルク制限を変更、省略可能
T3.14 3.0 2.0
```

`T`はトルク制御方法の表示または設定できます。

* 0 - 電圧
* 1 - 電流
* 2 - FOCアルゴリズムで計算した推定の電流

`E`はモーターへ電力供給をオンオフします。使わないときはオフにすることでバッテリーを節約できます。

* 0 - 無効
* 1 - 有効

# おわりに

次回、Teleplotでテレメトリをグラフ表示します。
