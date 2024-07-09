---
title: ジンバルモーター、SimpleFOCで動かす(5) - 磁気エンコーダAS5600
tags:
  - Arduino
  - ArduinoUnoR4
  - AS5600
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

[前回の](https://qiita.com/8ga3/items/ae252c0afe864eb960ae)の続きです。

今回使用した磁気エンコーダについてになります。

# AS5600

AliExpressでAS5600を購入しました。購入した理由は、SimpleFOCがデフォルトで対応しているからでした。

色々調べると、これを書いている時点では精度・入手性においてMT6701を買っても良さそうなので、後ほど試してみたいと思ってます。AS5600より少し高いくらいのようです。

購入したボードの写真です。

![IMG_6740.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/607701c2-e3f2-69fc-3fd2-1c5f6d756389.jpeg)

# 回路図

どんな回路図になっているかわからないので、表面のパターンとドキュメントのリファレンス回路図を参照しながら、自分で回路図を書いてみました。

![AS5600-Circuit.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/fab3c151-1560-0c3b-f4ca-3d305c19b5c1.png)

ドキュメントにはVDDが5Vのときと、3.3Vのときの回路が提示されています。

![AS5600-Ref.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e17ee90f-15e4-1e2c-9e64-965e0dd8d72e.png)

このボードはVDDが3.3Vになるように接続されていました。

# 5V対応に加工

私はArduino Uno R4 Minimaは5Vで使えると都合がいいわけです。

実は試しに3.3Vのところに、Minimaの5V出力を繋いでみました。壊れるということはなくI2Cで通信可能でした。ただやはりブートのタイミングによってはAS5600とI2Cで繋がらなくなりました。

やはりマニュアル通り5V設定に変えました。R1に0Ω抵抗がありますが、これを外すだけで5V設定になることがわかります。

チップ抵抗をはんだごて２本を使って、両端子に当てながらハンダを溶かして外しました。

![IMG_6741.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/55de7091-b900-8c15-77a6-31cc0b0f0113.jpeg)

あとで余分なハンダを吸い取って、ヤニ汚れをきれいにして完了です。

動作確認し、問題なく動作しました。

# おわりに

販売ページに回路図を書いておいてくれれば助かるのになと思いました。このくらいならさほど苦労はないですが、ちょっと面倒です。

次回、電流センサーを追加してみたいと思います。
