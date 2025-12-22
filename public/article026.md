---
title: 謎のラジコン用受信機のファームウェアを最新版にしてみた
tags:
  - ラジコン
  - ExpressLRS
private: false
updated_at: '2025-12-23T00:48:55+09:00'
id: 9db86931ee6fe1b8a3ca
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

この記事では、Aliexpressで購入したメーカー不詳の謎のラジコン用受信機のファームウェアを最新版にアップデートするまでの顛末を紹介します。この受信機はExpressLRSプロトコルをサポートしており、最新の機能や改善点を享受するためにファームウェアの更新が必要です。

# 発端

ある日ラジコンボートを作ってみたいと思いました。あまり知識はなかったのですが、まずはパーツ集めから始めました。秋葉原の電子部品店・ジャンク屋を巡り、街のラジコンショップやプラモ屋を訪ね、ヤフオクとメルカリをチェックします。そして毎日一度は見てしまうAliexpressを眺めていました。すると、ExpressLRS対応の受信機が非常に安価で売られているのを発見しました。 **価格はなんと約750円。** ラッキーと思い購入してみることにしました。

自作ドローンは一部を除きExpressLRSでそろえていて、対応する送信機を所有していたのです。ExpressLRSはオープンソースの長距離無線通信プロトコルであり、ドローンやラジコンの制御に広く利用されています。世界中で支持されているし、OSSというのがいいですよね。

購入したのがこちら

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/df63d819-0d60-4f34-beea-4b1be1f10e77.jpeg" alt="expresslrs_receiver.jpg" width="400"/>

パッケージにブランド名がかいてありましたがよくわからない・・・

調べてみると[CYCLONEというブランドの商品](https://ja.aliexpress.com/i/1005006162299424.html)と全く同じです。

そしてCYCLONEというブランドもまた謎なんです。

ファームウェアのバージョンは3.2とパッケージに書いてあります。現在（2025年12月）の最新版は3.6.2です。最新版にするのが嗜みでしょう。

そこからが苦難の始まりでした。

# 接続図

ExpressLRS受信機は、マイコンにESP8285やESP32を搭載していることが多いです。これらはWi-Fiを持っていますが技適がないため使用することができません。

そこで使うのがUSBシリアル変換アダプタです。今回は手元にあったCP2102搭載のものを使用しました。たぶんFTDIやCH340などでも大丈夫だと思います。

![diagram.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5ee87c2f-c556-48db-b710-aaa99f5dd41a.jpeg)

私の使ったUSBシリアルから5Vが出ていましたが、十分な電力が供給できなかったのか、うまく動作しませんでした。別途5Vの電源を用意することをお勧めします。4V～6Vの電源を供給してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/311e5230-3166-47ec-8160-04b45b9f3cdf.jpeg" alt="power_supply.jpg" width="400"/>

写真は私が接続した時の様子です。ESCの動作チェックをするつもりだったのと、ESCが5V出力を持っていたのでそこから電源を取りました。

基板の表側にボタンがあるので、これを押しながら電源を入れると **ブートローダーモード** になります。

:::note warn
ボタンを押さずに電源投入すると、接続したピンはPWMとして動作します。プロポとバインドできないまま60秒たつと、Wi-Fiアクセスポイントモードになります。多くは語りませんが注意してください。
:::

# ExpressLRS Configuratorでの設定とファームウェア書き込み

ExpressLRS Configuratorは、ExpressLRSのファームウェアを簡単に設定・書き込みできるツールです。
[公式サイト](https://www.expresslrs.org/)からExpressLRS Configuratorをダウンロード・インストールします。
ダウンロードせず、[Web Flasher](https://expresslrs.github.io/web-flasher/)を使う方法もあります。

![elrs_1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/802368d2-9621-4a75-8bd6-0cbbe2dadf08.png)

最新リリース版(2025年12月時点でv3.6.2)を使用しました。

ターゲットデバイスのカテゴリは **Happymodel 2.4GHz** 、デバイスは **Happymodel EPW6 2.4GHz PWM RX** を選択します。なぜなら、この受信機はHappymodelのEPW6 2.4GHz PWM RXとピン配置が同じだからです。[^yotube] [^issue]

:::note warn
過去のバージョンではカテゴリは **Generic targets used as base 2.4 GHz** 、デバイスは **Generic ESP8285 7xPWM 2.4GHz RX** を選択するように書かれていることがありますが、最新版では上記の選択肢が適切です。
かつて[DIY Receiver](https://www.expresslrs.org/hardware/special-targets/diy-rx/)のためのターゲットとして存在していました。これを製造業者が勝手に利用していたようです。これらの業者はExpressLRSプロジェクトに対して非協力的であり、対応に苦慮していたようです。Generic targetsは廃止に至りました。
:::

![elrs_2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/69c9d38e-2a7a-4ca8-a705-0872a972e889.png)

シリアルデバイスを選択し、**フラッシュ** ボタンをクリックします。接続が成功すると、ファームウェアの書き込みが開始されます。

私は、 **フラッシュの前に消去** オプションを有効にしました。

:::note info
消去せずに書き込んだところ、送信機とバインドできなかったためです。
:::

[^yotube]: [ELRS Tx Rx "SECRET" Hardware Tab CYCLONE ELRS 7CH PWM JSON Files](https://www.youtube.com/watch?v=2VZb0n_G_Pg)
[^issue]: [Generic targets missing in Configurator? #2726](https://github.com/ExpressLRS/ExpressLRS/discussions/2726)


# 7CHを使えるようにする

ExpressLRS Configuratorでファームウェアを書き込んだ後、7CHを使用するためには追加の設定が必要です。

受信機に電源投入後60秒待つとブルーLEDがそれまでより速く点滅します。Wi-Fiアクセスポイントモードになったことを示しています。[^wifi]

ホットスポット **ExpressLRS TX** に接続します。パスワードは **expresslrs** です。ブラウザで [http://10.0.0.1/hardware.html](http://10.0.0.1/hardware.html) にアクセスします。

**PWM output pins** の欄に **0, 1, 3, 9, 10, 5, 16** と入力します。SAVEボタンを押してリブートすると7CHが有効になります。

基板にLEDと書いてあるのですが、このjsonファイルのledというキーには16とあります。pwm_outputsに16を追加することで7CH目が有効になるわけです。

または下記のJSON設定ファイルをダウンロードし、上記のページでインポートしても同様の効果があります。

```json
{
    "customised": "true",
    "serial_rx": -1,
    "serial_tx": -1,
    "radio_dio1": 4,
    "radio_miso": 12,
    "radio_mosi": 13,
    "radio_nss": 15,
    "radio_rst": 2,
    "radio_sck": 14,
    "power_min": 0,
    "power_high": 0,
    "power_max": 0,
    "power_default": 0,
    "power_control": 0,
    "power_values": [13],
    "led": 16,
    "pwm_outputs": [0, 1, 3, 9, 10, 5, 16],
    "vbat": 17,
    "vbat_offset": 12,
    "vbat_scale": 310
}
```

[^wifi]: [Wifi Updating - ExpressLRS](https://www.expresslrs.org/software/updating/wifi-updating/)

# CRSFモードに変更する

ExpressLRS受信機はデフォルトでPWMモードで動作しますが、CRSF（Crossfire）モードに変更することも可能です。

前項と同様にWi-Fiアクセスポイントモードに入り、下記のJSON設定ファイルをダウンロードし、インポートします。

```json
{
    "customised": "true",
    "serial_rx": 3,
    "serial_tx": 1,
    "radio_dio1": 4,
    "radio_miso": 12,
    "radio_mosi": 13,
    "radio_nss": 15,
    "radio_rst": 2,
    "radio_sck": 14,
    "power_min": 0,
    "power_high": 0,
    "power_max": 0,
    "power_default": 0,
    "power_control": 0,
    "power_values": [13],
    "led": 16,
    "pwm_outputs": [-1],
    "vbat": 17,
    "vbat_offset": 12,
    "vbat_scale": 310
}
```

# 内部の仕組み

どうして **Happymodel EPW6 2.4GHz PWM RX** を選択するのか気になる方もいるでしょう。

ExpressLRS Configuratorは、GitHubのリポジトリ[ExpressLRS/targets: ExpressLRS target hardware layout files](https://github.com/ExpressLRS/targets)にある[targets.json](https://github.com/ExpressLRS/targets/blob/master/targets.json)を参照しているようです。

従来まで選択していた[Generic ESP8285 6xPWM 2.4Ghz RX](https://github.com/ExpressLRS/targets/blob/3a4bdba5879ff86272af185d04100793c051ed07/targets.json#L1458)を見つけることができます。

```json
            "pwm6": {
                "product_name": "Generic ESP8285 6xPWM 2.4Ghz RX",
                "lua_name": "ELRS+PWM 2400RX",
                "layout_file": "Generic 2400 PWMP6.json",
                "upload_methods": ["uart", "wifi"],
                "min_version": "3.0.0",
                "platform": "esp8285",
                "firmware": "Unified_ESP8285_2400_RX",
                "prior_target_name": "DIY_2400_RX_PWMPEX"
            },
```

そこで先程選択した[Happymodel EPW6 2.4GHz PWM RX](https://github.com/ExpressLRS/targets/blob/3a4bdba5879ff86272af185d04100793c051ed07/targets.json#L1981) を見てみましょう。

```json
            "epw6": {
                "product_name": "HappyModel EPW6 2.4GHz PWM RX",
                "lua_name": "HM EPW6 2400",
                "layout_file": "Generic 2400 PWMP6.json",
                "upload_methods": ["uart", "wifi"],
                "min_version": "3.3.0",
                "platform": "esp8285",
                "firmware": "Unified_ESP8285_2400_RX",
                "prior_target_name": "DIY_2400_RX_PWMPEX"
            },
```

比べるとほぼ一緒ですね。 **layout_file** と **firmware** が同じであるところがポイントです。

[Generic 2400 PWMP6.json](https://github.com/ExpressLRS/targets/blob/master/RX/Generic%202400%20PWMP6.json)を見てみます。

[前項](#7chを使えるようにする)のjsonファイルと似ていることが分かります。

```json
{
    "radio_dio1": 4,
    "radio_miso": 12,
    "radio_mosi": 13,
    "radio_nss": 15,
    "radio_rst": 2,
    "radio_sck": 14,
    "power_min": 0,
    "power_high": 0,
    "power_max": 0,
    "power_default": 0,
    "power_control": 0,
    "power_values": [13],
    "power_lna_gain": 0,
    "led": 16,
    "pwm_outputs": [0,1,3,9,10,5],
    "vbat": 17,
    "vbat_offset": 12,
    "vbat_scale": 310
}
```

このように設定を追いかけると、なぜこのターゲットを選択するのか理解できると思います。自前でファームウェアをビルドしてフラッシュする手間を省けるかもしれませんね。

# まとめ

Aliexpressで購入した謎のExpressLRS対応受信機のファームウェアを最新版にアップデートする方法を紹介しました。ExpressLRS Configuratorを使用することで、簡単にファームウェアの書き込みと設定が可能です。7CHの有効化やCRSFモードへの変更も簡単に行えます。

ここでは紹介しませんでしたが、テレメトリとして電圧を送信することも可能です。（日本国内不可）

次に受信機を購入するときは、公式にサポートされている製品を選ぶようにしたいと思います。

ExpressLRSのオープンソースコミュニティに感謝しつつ、快適なラジコンライフを楽しんでください。

:::note warn
この受信機は技適がないので、日本国内では使い方によって違法になります。この記事は情報提供のみを目的としており、違法行為を助長するものではありません。
:::
