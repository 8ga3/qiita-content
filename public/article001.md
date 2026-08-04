---
title: CH32V003F4P6をch32v003funを使ってLチカするまで (macOS)
tags:
  - WCH
  - RISC-V
  - CH32V003
  - CH32V003F4P6
private: false
updated_at: '2023-10-03T22:08:23+09:00'
id: 76a0f428c710875dffeb
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

<img width="540" alt="CH32V003F4P6" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f5f78427-e83f-0783-8324-6deaed802861.jpeg">

Intel MacのmacOS 13.6を使用します。

こちらの説明通りに作業を進めました。
https://74th.hateblo.jp/entry/ch32v003fun

# コンパイラが見つからない

`./ch32v003fun/ch32v003fun/ch32v003fun.mk`を開発環境に合わせてあげます。

```diff_make:ch32v003fun.mk
- PREFIX?=riscv64-unknown-elf
+ PREFIX?=riscv32-unknown-elf
```

# Build

```shell
% make all
riscv32-unknown-elf-gcc -o test.elf ch32v003fun/ch32v003fun/ch32v003fun.c test.c -g -Os -flto -ffunction-sections -static-libgcc -march=rv32ec -mabi=ilp32e -I/usr/include/newlib -I./ch32v003fun/ch32v003fun/../extralibs -I./ch32v003fun/ch32v003fun -nostdlib -I. -Wall  -L/usr/local/opt/bison/lib -T ./ch32v003fun/ch32v003fun/ch32v003fun.ld -Wl,--gc-sections -L./ch32v003fun/ch32v003fun/../misc -lgcc
dyld[34374]: Library not loaded: /usr/local/opt/isl/lib/libisl.22.dylib
  Referenced from: <97D19A7B-A41C-3329-A4F2-A3DDEE64C664> /opt/riscv/libexec/gcc/riscv32-unknown-elf/8.3.0/cc1
  Reason: tried: '/usr/local/opt/isl/lib/libisl.22.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/usr/local/opt/isl/lib/libisl.22.dylib' (no such file), '/usr/local/opt/isl/lib/libisl.22.dylib' (no such file), '/usr/local/lib/libisl.22.dylib' (no such file), '/usr/lib/libisl.22.dylib' (no such file, not in dyld cache), '/usr/local/Cellar/isl/0.26/lib/libisl.22.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OS/usr/local/Cellar/isl/0.26/lib/libisl.22.dylib' (no such file), '/usr/local/Cellar/isl/0.26/lib/libisl.22.dylib' (no such file), '/usr/local/lib/libisl.22.dylib' (no such file), '/usr/lib/libisl.22.dylib' (no such file, not in dyld cache)
riscv32-unknown-elf-gcc: internal compiler error: Abort trap: 6 signal terminated program cc1
```

エラーで止まりました。
`libisl.22.dylib`がないとでています。

# libを誤魔化す

私の環境にインストールされているlibが`libisl.23.dylib`だったので、`libisl.22.dylib`としてシンボリックリンクを作成します。バージョンが一つしか違わないので多分動くでしょう😜

```shell
% cd /usr/local/Cellar/isl/0.26/lib/
% ln -s libisl.23.dylib libisl.22.dylib
% cd -
```

# 再Build ＆ Flush

```shell
% make all
... 中略 ...

Found WCH Link
WCH Programmer is LinkE version 2.10
Chip Type: 003
Setup success
Flash Storage: 16 kB
Part UUID    : 5c-96-ab-cd-c4-b6-bc-53
PFlags       : ff-ff-ff-ff
Part Type (B): 07-13-bb-91
Read protection: disabled
Interface Setup
Image written.
```

# 書き込み成功🎉

おつかれさま🍺

![Lチカ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/be20ccd4-4d66-365b-0c0c-3a93e6fc2b7c.jpeg "Blink")
