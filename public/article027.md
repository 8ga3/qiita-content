---
title: 100均タッパーのラジコン船をEdgeTXミキシングでドローン風に操作する
tags:
  - ラジコン
  - EdgeTX
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

ダブルモーターのラジコン船を、EdgeTXプロポのミキシング機能でドローンと同じ感覚で操作できるようにする方法を紹介します。

# ラジコン船の構成

百均のタッパーを使ってラジコン船を作りました。こういう船は世界中で作られていると思います。タンク型ラジコンと同じ仕組みでシンプルです。

![Cheap-Ship.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3594bd61-58a7-463d-9718-95540efe7e66.jpeg)

機能ブロック図はこんな感じです。

![機能ブロック図](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/54eb4434-3209-4905-96a7-c6dcb0ffd120.png)

# 課題と方針

タンク用モーターESCだと、左右のモーターを独立して操作することになります。慣れていないと操作しづらい。ドローンと同じようにスロットルとエルロンで操作したいところです。

うまい具合にモーターを制御してくれるESC、たとえば近藤科学の[MD-2](https://kondo-robot.com/product/40454)もありますが、ちょっと高い。Pixhawkのようなフライトコントローラーを使う手もありますが、これも高価です。

なるべく安く済ませたいので、AliExpressで安いダブルモーターESC [KINGMODEL MIXDRI PRO 5AX2](https://ja.aliexpress.com/item/33008316502.html)を購入しました。送信機のミキシング機能で操作しやすくすれば解決です。

実はこのESCにはスイッチで独立モードとミキシングモードを切り替える機能がありました。いろいろ設定し始めてから気がついたのですが、悔しいので最後まで続けました。😓

ついでに、ドローンと同じようにアーム・ディスアームもスイッチで操作できるようにしていきます。

# 使用機材

送信機はJumper T-Lite V1 (JP4IN1)で、ちょっと古めのモデルです。もともとOpenTXでしたが、EdgeTXに書き換えています。外部モジュールにBETAFPV ELRS Nano TX Moduleを搭載しています。

- EdgeTX Version 2.11.5（Jumper T-Liteは2.12以降は非対応）
- ExpressLRS Version 3.6.2（V4.xは気が向いたらアップデートする）

受信機は、[以前の記事](http://127.0.0.1:8888/items/9db86931ee6fe1b8a3ca)で取り扱った[ELRS 2.4Ghz PWM 7CH](https://ja.aliexpress.com/item/1005007033545569.html)です。

# 設定手順

文章で書くと長くなるので、EdgeTX Companion 2.11.5 のスクリーンショットで説明していきます。

## セットアップ

外部送信モジュールを使用する設定です。
ELRS 3.xはアームロック解除がCH5固定です。

![etx-setup.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/887ef299-e936-4144-9565-0e69e8e544d8.png)

## 入力

Jumper T-LiteはSAとSBが3ポジション、SCとSDが2ポジションスイッチです。
アームロック解除は1bitなので、2ポジションスイッチを割り当てます。
3ポジションでもいけますが、中央の位置が無意味になるのでもったいないですね。
ここではSDをCH5に割り当てています。

![etx-inputs.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/99e02723-1e09-4439-88cb-e665d65d3878.png)

## 出力

名前をつけているくらいで、特に変更はしていません。

![etx-outputs.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1deff188-1990-4441-84b6-ee3336f1297b.png)

## カーブ

CV1〜4に設定を入れています。ミキシングで使います。

![etx-curves.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/97126c75-9e98-4570-bbec-310d67c02487.png)
![etx-cv1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3c903956-c5f9-4d14-a68b-f8995a1e44e8.png)
![etx-cv2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5bbd254a-e9f6-463d-9a95-bcf9d3307950.png)
![etx-cv3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1354de70-1794-464f-9270-c3f4bb413b92.png)
![etx-cv4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1ef373b3-6383-4d04-95b9-a5fa66d39dd8.png)

## 論理スイッチ

L01~L05に設定しています。ミキシングに使用します。

![etx-logicalswitch.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/c7c2d51f-2696-477e-9e09-75b07641854e.png)

## ミキシング

スクリーンショットの通り、かなり複雑です。
説明し始めると終わらないので、設定画面を全部載せておきます。

![etx-mixes.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/cf0a97e5-482d-4435-976d-0bcb7a1659ed.png)

### CH1
#### CH1-1
![m-ch1-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/cc1ec70e-2a0d-444f-a0cb-d8e46d9de118.png)
#### CH1-2
![m-ch1-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bbf31cc3-1996-413e-be99-95b7e39a3a48.png)

### CH2
#### CH2-1
![m-ch2-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bb98b8f1-52e3-419c-9f75-bc1e4bf81fdd.png)
#### CH2-2
![m-ch2-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/30545da1-b197-42b0-b5df-6eb50b82ba80.png)

### CH5
![m-ch5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bad7409a-6597-45bb-aa27-416d80cbec7a.png)

### CH6
![m-ch6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ec8fb80c-47de-40f0-a611-cf5be6cddb70.png)

### CH7
![m-ch7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d048f5b7-3b9d-469e-8138-0fd9861382ba.png)

### CH8
![m-ch8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bcd5655c-d343-4253-b33a-40bfdc1ff75a.png)

### CH11
#### CH11-1
![m-ch11-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/891aeeff-a14e-49e6-99b5-c32cb834fdaf.png)
#### CH11-2
![m-ch11-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/01128427-d478-419e-8f7c-bfdbbbd9fbaa.png)
#### CH11-3
![m-ch11-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/4f6e06a2-88de-4805-89ab-c61824d4013d.png)

### CH12
#### CH12-1
![m-ch12-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d234c334-42d3-463d-b55c-be77155ca0cf.png)
#### CH12-2
![m-ch12-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3da7574b-5476-45ab-ac6d-9adc98d642a7.png)
#### CH12-3
![m-ch12-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b9280850-c26a-488d-80da-128168914902.png)

### CH13
#### CH13-1
![m-ch13-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/0845ff81-35d4-4b20-8ab5-e9f971565bf9.png)
#### CH13-2
![m-ch13-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f2946f50-a0be-4d80-8174-0b4d054727c6.png)
#### CH13-3
![m-ch13-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/0eea3e23-2853-4e55-b104-dbf62c3bda27.png)

### CH14
#### CH14-1
![m-ch14-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bab259f6-02cf-41af-b6d9-d432af551e8b.png)
#### CH14-2
![m-ch14-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d91efd53-3b1e-4458-a220-4fa9f78e981d.png)
#### CH14-3
![m-ch14-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/282b2b57-f8ec-490e-85a3-1e2f0a53bc0f.png)

## ミキシング・フローチャート

ミキシング画面だけだとさっぱり分からないので、フローチャートを作りました。自分自身の理解にも役立ちました。

![edgetx-diagram.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a0578a43-e9d0-491e-8bc7-5f9068f7898b.png)

# 動作確認

## アームロック

SDスイッチがOFFの状態です。
プロポの電源を入れたときにSDスイッチがONだったとしても、いきなり動かないようにミキシングで制御しています。

![radio-off-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/07570755-6b45-4490-b95c-65a812c0e444.png)

スティックを動かしても、出力に変化はありません。

![radio-off-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a76dd791-225a-4972-9737-50aedd851a9f.png)

## アームロック解除

![radio-on.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5ce81b19-6581-48db-a8a8-fdf6fc464585.png)

## 前進

![radio-on-f.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/8a80d99f-48c8-49bd-aa84-c3cd085e2d49.png)

## 前進右

エルロン（モード2では右スティックの横方向）を右に入れると、左モーターの出力が下がります。
出力ゼロ（PWM 1500）付近だと不安定になるため、一定以下には下がらないように制御しています。

![radio-on-f-r.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3629e73c-dedf-496f-b85a-9018ebb8dace.png)

## 前進左

エルロンを左に入れると、右旋回と逆の動作になります。

![radio-on-f-l.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/10c41eb4-8883-4343-b56c-c2f77a0312f1.png)

## 後退

![radio-on-b.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/85da3114-b480-4417-b599-7774a015ea74.png)

## 後退右

![radio-on-b-r.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3ee52e2d-ec6d-418c-80f6-24e88412b9ce.png)

## 後退左

![radio-on-b-l.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1febd295-7dce-45a2-887d-4d8bf26940e9.png)

## 回転右

位置はそのままで右回転します。

![radio-on-rot-r.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ece3409d-7499-456b-b9dd-0f4dac542910.png)

## 回転左

位置はそのままで左回転します。

![radio-on-rot-l.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a558fbd5-4e7b-4a1a-b6f6-b6c6ca41bb91.png)

# 航海

公園の噴水でいざ出港。
ドローンと同じ感覚で操作できるようになったので大満足です。

![sail.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b5226103-a15a-4f21-b54e-56675e31efbd.gif)

# まとめ

EdgeTXのミキシング設定について、以前より理解が進んだと思います。
まだまだ触れていない機能が多いので、何かの折に試してみたいところです。

次はArduPilotで自動航行にも挑戦してみたいですね。

# 参考

[YouTube動画 【キャタピラ車でドリフト】プロポ設定を完全解説！自作ラジコン戦車](https://www.youtube.com/watch?v=FX6u_qRm0h8)

[Motor arming/kill switch](https://rc-soar.com/edgetx/setups/motor_arm/motor_arm.php)
