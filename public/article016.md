---
title: ジンバルモーター、SimpleFOCで動かす(6) - 電流センサ ACS712追加
tags:
  - Arduino
  - PlatformIO
  - BLDC
  - SimpleFOC
  - ArduinoUnoR4
private: false
updated_at: ''
id: 859013d7fd19154372ec
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/eaa26381e1a4a7a6b8da)の続きです。

電流センサ **ACS712** を追加します。

# ACS712について

SimpleFOCは、基本的に電流センサがなくてもモーターは指示したとおりに動いてくれます。センサがあればもっと精度がよくなるのでしょう。閉ループ制御を謳った業務用モジュールでは、多くは電流センシングできるようです。というわけで電流センサを導入しました。（速度起電圧を利用することでセンサレスにできるそうです。）

まずは[SimpleFOCShield v3.2](https://docs.simplefoc.com/arduino_simplefoc_shield_showcase)が採用しているACS712にしました。AmazonやAliExplessなどで購入できるため入手性もよく、一個200~400円くらいで安くて助かります。スタートとしてはよいのではと思います。

![IMG_6676.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ff59c773-2038-6c5a-eb26-ac9dbf36dba9.jpeg)

購入時の注意点として、多くのチップと同様にACS712はバリエーションがあります。計測できる電流最大値が5A、20A、30Aがあります。つかっているブラシレスモーターは5Aで十分です。20A、30Aでも使えないことはありませんが、分解能の低下が懸念されます。ちなみに私は間違えて20Aを買ってしまって、あとから5Aを買い直しました。

写真のように、２列目に **ELC-05B** と印字されているかで確認できます。

![IMG_6733.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/4ba869ae-ba31-19bc-fd7d-893a96c7df22.jpeg)

今回は３相ブラシレスモーターを１つにつき電流センサは２個使います。３相なので３つ使うこともできますが、理論的には２個あればよいそうです。

# 接続

モーターにつながる電流する方法が３つあります。

* ハイサイド電流センシング
* ローサイド電流センシング
* インライン電流センシング

それぞれの仕組みについては[ドキュメント](https://docs.simplefoc.com/current_sense)を参照してもらえればと思います。SimpleFOCMiniではインライン電流センシングのみ可能です。

接続場所は、モーター線M1・M2・M3のうち２本を使い、間にセンサを入れます。

![IMG_6744.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/718338d2-77c2-c1da-5084-8789089679b4.jpeg)

RCサーボでよく使うJR(日本遠隔制御)配色の３線ケーブルで製作しました。両端にQIコネクタをつけます。

![IMG_6727.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e349d96b-7925-6aec-9406-13ac80b06aa2.jpeg)

ACS712を取り付けるため、３線の途中を剥がし切断し被覆を剥きます。２個の電流センサの干渉をさけるため、切断位置を少しずらしています。

![IMG_6728.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d58c7031-2fdb-a7ba-14b8-7940153ead47.jpeg)

つぎにセンサ出力と電源を接続するケーブルを作成します。VCCは5Vです。Arduino Uno R4 Minimaには5V出力ソケットが１箇所しかありません。磁気センサにも電力供給しなくてはいけませんので、5Vを３つに分岐させる必要があります。

私は分岐ケーブルを自作することにしました。BECを作るとか、ブレッドボードを使う手も考えましたが、今回はケーブルのほうが邪魔じゃなくていいかなと思いました。

こんな感じで作りました。赤5V、黒GND、白が電流センシングしたアナログ値になります。

![IMG_6726.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d8e5021d-e1e3-52b0-e423-f719c7c6e4b0.jpeg)

赤・黒は5VとGNDにつなぎます。白はANALOG INのA0とA1につなぎました。モーター線のM1・M2の順番どおりに接続しましょう。間違えたらモーターが動きませんでした。

![IMG_6735.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/2e977c0c-f89d-77c1-a7e9-2a51ef384207.jpeg)

# コーディング

グローバルにインスタンスを作成します。

先に引数の２番目、３番目はSimpleFOCMiniのコネクタM1、M2につなげた電流センサのOUTにつながるアナログイン番号を指定します。

電流センサを３つつけたときは、`A0, A1, A2`のように書きます。デフォルトは`_NC`で未使用という意味になります。

```cpp
InlineCurrentSense current_sense = InlineCurrentSense(185.0f, A0, A1);
```

引数１番目はA(アンペア)あたりのmVです。ACS712データシートによるとTpy. 185 mV/Aとあるので、そのまま185.0fを指定します。

![acs712spec.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ac6f06db-738f-b28f-b245-e332cf8c9f1a.png)

グラフを見ると、0Aのとき2.5Aが出力されます。実際には誤差があるので、current_sense.init()で初期化するときに0Aのときの測定値を使います。

-5Aから+5Aまできれいな比例になっています。

温度による変化は小さそうなので無視してよさそうです。

![acs712_vol_vs_cur.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/cf1dfa46-790e-7806-56b6-f69969268788.png)


セットアップを行います。`BLDCMotor::init()`をコールする前に`BLDCMotor::linkCurrentSense()`で登録を済ませます。

```cpp
void setup() {
  ...

  current_sense.linkDriver(&driver);
  current_sense.init();
  motor.linkCurrentSense(&current_sense);

  motor.init();
  motor.initFOC();

  ...
}
```

loop関数では特にやることはありません。ライブラリ内部で実行されます。

[ジンバルモーター、SimpleFOCで動かす(4) - Teleplotでグラフ表示](https://qiita.com/8ga3/items/ae252c0afe864eb960ae)で紹介したTeleplotで電流センサ値を表示する場合は、`REG_CURRENT_A, REG_CURRENT_B`を登録します。

```
  uint8_t reg[] = { REG_TARGET, REG_ANGLE, REG_VELOCITY, REG_CURRENT_A, REG_CURRENT_B };
  telemetry.setTelemetryRegisters(5, reg);
```

または、メンバ関数でも値を取ってこれるので、シリアル出力などお好みに合わせて利用できます。

```cpp
  PhaseCurrent_s currents = current_sense.getPhaseCurrents();
  float current_magnitude = current_sense.getDCCurrent();
```

# Renesas RA4M1のADC14bit読み込み

記事を書いている2024年7月時点の[マニュアル](https://docs.simplefoc.com/inline_current_sense)によると、Arduino Uno R4は未対応(TDB)となっています。とはいえ、Arduinoライブラリがハードウェアの違いを吸収してくれるのでアナログ値を問題なく読み込むことができます。

この場合、ADCは10bit精度、つまり0~1023の値になります。

Arduino Uno R4のRenesas RA4M1は、ADCを最大14bitで読むことができます。せっかくなら使いたいところです。

ビット数の変更は簡単で、`analogReadResolution(14)`とコールするだけで変更できます。

InlineCurrentSenseクラスが内部でハードウェア依存部分としてコールする`_configureADCInline()`をコーディングします。

なお、インライン電流センシングのみになります。ローサイドのときは同期するためにタイマ割り込みするので、さくっとコーディングとはいきません。

動作確認するとちゃんと動いてくれました。14bitの分解能が活きているような気はしないですけど😅

```cpp
#include <current_sense/hardware_api.h>

#if defined(ARDUINO_UNOR4_WIFI) || defined(ARDUINO_UNOR4_MINIMA)

#define _ADC_VOLTAGE 5.0f
#define _ADC_RESOLUTION_BIT 14
#define _ADC_RESOLUTION 16384.0f

// function reading an ADC value and returning the read voltage
void* _configureADCInline(const void* driver_params, const int pinA,const int pinB,const int pinC){
  _UNUSED(driver_params);

  analogReadResolution(_ADC_RESOLUTION_BIT);

  if( _isset(pinA) ) pinMode(pinA, INPUT);
  if( _isset(pinB) ) pinMode(pinB, INPUT);
  if( _isset(pinC) ) pinMode(pinC, INPUT);

  GenericCurrentSenseParams* params = new GenericCurrentSenseParams {
    .pins = { pinA, pinB, pinC },
    .adc_voltage_conv = (_ADC_VOLTAGE)/(_ADC_RESOLUTION)
  };

  return params;
}
#endif
```

# おわりに

次回、SimpleFOCStudioについて書いてみたいと思います。

https://qiita.com/8ga3/items/29c241aa43adbf6908d1
