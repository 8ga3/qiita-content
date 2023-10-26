---
title: ch32v003funのexamples/i2c_oledで、8ピクセル右にズレて描画されることがある問題
tags:
  - WCH
  - RISC-V
  - CH32V003
  - ch32v003fun
private: false
updated_at: '2023-10-26T19:15:40+09:00'
id: ad895af66967627a1fc2
organization_url_name: null
slide: false
ignorePublish: false
---
# 症状

[ch32v003fun](https://github.com/cnlohr/ch32v003fun)を使ってOLEDに画像を表示しようと思い、examplesを参考にしました。i2c_oledという、ずばりのものがあります。最初は気が付かなかったのですが、左側の隙間がちょっと広いときがあります。よく調べてみると、`ssd1306_drawImage()`関数で画像を描画すると、8ピクセルだけ右にズレた位置から描画が始まってました。

# 対処方法

[examples/i2c_oled/ssd1306.h](https://github.com/cnlohr/ch32v003fun/blob/f1820f2995e146ce8be0e3a94cb34a53d8238061/examples/i2c_oled/ssd1306.h#L268C39-L268C39)を修正します。アドレス計算が間違っていました。

```diff_c
--- a/examples/i2c_oled/ssd1306.h
+++ b/examples/i2c_oled/ssd1306.h
@@ -265,7 +265,7 @@ void ssd1306_drawImage(uint8_t x, uint8_t y, const unsigned char* input, uint8_t
                        uint8_t input_byte = input[byte + line * bytes_to_draw];
 
                        for (pixel = 0; pixel < 8; pixel++) {
-                               x_absolute = x + 8 * (bytes_to_draw - byte) + pixel;
+                               x_absolute = x + 8 * (bytes_to_draw - 1 - byte) + pixel;
                                if (x_absolute >= SSD1306_W) {
                                        break;
                                }
```

# おわりに

Issueを出しましたが今のところ反応がないですね。
