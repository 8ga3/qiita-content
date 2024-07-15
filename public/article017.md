---
title: ジンバルモーター、SimpleFOCで動かす(7) - SimpleFOCStudio編
tags:
  - Arduino
  - BLDC
  - SimpleFOC
  - SimpleFOCStudio
  - ArduinoUnoR4
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/859013d7fd19154372ec)の続きです。

SimpleFOCStudioについてになります。

![std1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5f01a133-55d5-4272-61f7-48531e0d40ff.png)

# SimpleFOCStudioについて

これまでモーターの動作確認やPID等の調整は、スケッチを直接書き換えてビルド・アップロードをするか、SimpleFOCのCommanderを使ってシリアルモニターからコマンドを入力して調整するかでした。グラフはTeleplotを使って表示していました。

SimpleFOCStudioは、すべての操作をGUIから操作でき同時にグラフ表示が可能で、とてもユーザーフレンドリーです。調整が終わったらスケッチを生成してくれます。

# インストール

GitHubに[プロジェクト](https://github.com/JorgeMaker/SimpleFOCStudio)があります。インストール方法が書いてあります。ここでは私が実施した手順を記載していきます。

メインブランチのヘッドをクローンしました。リリースバージョンはちょっと時間が経っているようです。

ソースコードをダウンロードするかgitコマンドを使います。

```shell
% git clone https://github.com/JorgeMaker/SimpleFOCStudio.git
% cd SimpleFOCStudio
```

Pythonで作成されているのでインストールしておきます。ドキュメントにはPython 3.9を要求していますが、3.10と3.11でも動作しました。3.12はパッケージのインストールでエラーになったので避けたほうがよさそうです。私は3.11を使うことにしました。

```shell
% python3.11 -V
Python 3.11.9
```

venvでSimpleFOCStudio専用の環境を作成し、pipでパッケージをインストールします。

```shell
% python3.11 -m venv .venv
% source .venv/bin/activate
% pip install -r requirements.txt
```

アプリを実行します。

```shell
% python simpleFOCStudio.py
```

SimpleFOCStudioを終了したときは、`deactivate`コマンドで専用環境から抜けます。ターミナルを閉じてもよいです。

```shell
% deactivate
```

# スケッチ

コールバック関数を定義します。メンバ関数`motor()`によって、SimpleFOCのすべてのパラメータの操作が可能となります。`motion()`は使わないので注意しましょう。

```cpp
Commander command = Commander(Serial);
void doMotor(char* cmd) { command.motor(&motor, cmd); }
```

セットアップ関数で初期化します。`monitor_downsample`は0にしておきます。0ですとシリアルポートには表示されないですが、Commanderを通してGUIで指定できるため問題ありません。

`add()`でコールバック関数を登録します。最初の引数`M`の文字は何でもよいですが、アプリ側で設定で合わせることになります。

```cpp
void setup() {
  ...
  motor.useMonitoring(Serial);
  motor.monitor_downsample = 0;

  ...

  command.add('M', doMotor, "motor");
  ...
}
```

ループ関数で`monitor()`をコールします。

```cpp
void loop() {
  motor.loopFOC();
  motor.move();
  motor.monitor();
  command.run();
}
```

# アプリの使い方

ウインドウが表示されたら、左上のモーターのアイコンをクリックします。ポップアップメニューが表示されます。**Tree View** と **Form View** が選べます。 **Tree View** のほうが細かい設定ができるようになってます。 **Tree View** の操作について説明しますが、 **Form View** もそれほど違いはないので、使い方に迷うことはないと思います。

![std2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/4aa3fd6d-b336-8fa0-5a6f-0e9f9d3a432d.png)

Arduino Uno R4とはUSBで接続しておいてください。矢印のところの **Configure** をクリックします。

![std3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/34e5b6b1-8e83-9291-1486-820cced5e0f6.png)

シリアルポートを指定します。大抵は **Port Name** と **Bit rate** を設定するくらいだと思います。

![std4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/09355ee8-469d-a2c9-ee66-dfc25204a30b.png)

①のところにスケッチ`add()`関数の最初の引数で指定した文字を入力します。

②のConnectをクリックすると、Arduino Uno R4と接続開始します。

**Pull Params** ボタンは押さなくても、接続の最初に自動的にパラメータを取得してくれます。もし **Command Line interface** で手動設定したあとにGUIのほうを反映させるのに使えます。

![std5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a32d2f2c-1886-7aaa-f59c-1d093cdeb496.png)

間違いがなければArduinoと接続開始します。グラフは **Real time motor variables** の **Start** ボタンをクリックします。

![std6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ee74bcd8-8adc-5b5c-5cc1-46e8778c8752.png)

**Angle** は今の角度（ラジアン）、 **Velocity** は速度、 **Voltage** は電圧です。

**Target** は現在のモードでの目標値です。 **Motion Control Type** によって意味が変わるので注意が必要です。モードごとに目標値を持っているようです。

![std12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/c115c105-fc11-c1bb-6b0a-748b993aff90.png)

**Disable Device** はモーターへの電力供給を切ります。トグル操作なので **Disable Device** と **Enable Device** と表示が切り替わります。

**Sensor Zero** は、いまの値を0として絶対値の値をオフセットになります。図のように **Angle** が13.796...のときにボタンをクリックすると、**Angle** が0になり **Zero Angle Offset** は13.796...になります。磁気センサーの場合、モーターに取り付けた磁石の極と磁気センサーを頑張って合わせるより、オフセット値を調整したほうが楽かなと思います。

**Zero Target** は、ターゲットにゼロをセットします。Angleモードのときは角度0になるまでモーターは回転します。

![std13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/9219fae7-c948-eba5-c736-3657ed958da4.png)

ターゲットを変更します。 **Increment** の数値だけターゲットをプラス・マイナスします。>>はIncrementの二倍です。

![std14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bbf85c1a-c7a2-447f-b21a-2b34e9f66337.png)

スケッチを生成するボタンは左上の矢印の位置になります。

![std10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5a9733ac-d98a-182c-fe11-d40ee1827d77.png)

ダイアログボックスが表示されるので、必要と思う項目にチェックをいれてOKを押すとスケッチが表示されます。テキストボックスを選択してコピペしましょう。

![std7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e75773e2-79e5-addb-95c9-94128c35044c.png)

![std8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ff78f42b-f226-2416-491d-ae7d221f308d.png)

コマンドライン・インターフェースはDevice画面に表示されていますが、大きな画面で表示したいときは画面左上の矢印の位置になります。

![std11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1f0a3316-9b75-7662-a6f9-6180afc66958.png)

![std9.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/de71fda1-0450-e24a-4aad-999639bb5632.png)

# さいごに

ツリーに表示されたPID・フィルター・リミットを編集できるので、根気でチューニングしましょう。頑張ってください！
