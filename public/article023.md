---
title: Raspberry Pi 5 M.2 HAT+で単独ブートできないNVMe SSDをルートファイルシステムにする方法
tags:
  - RaspberryPi
private: false
updated_at: ''
id: 0e539134f0381d524e2f
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

先日、イベント特価になってた[Raspberry Pi 5 M.2 HAT](https://ssci.to/9248)を購入しました。

HATだけ持っていてもしかたないので、Toshiba KIOXIAのNVMeであるKBG40ZNS256Gの中古品を購入しました。そんなに使う目的も考えてなかったので、とりあえず安い品を選びました。それが失敗のもとで、ネットのあらゆる情報を参考にしても、うまくいきません。そもそもKBG40ZNS256Gでは、単独ブートできないと相談している人が世界中にいる状況です。😭

もったいないのでこのHATを使って、単独ブートできないNVMe SSDをRaspberry Pi 5のルートファイルシステムにする方法を紹介します。

:::note info
KBG40ZNS256GのFirmwareはさわっていません。
:::

# NVMe SSDのSMART情報確認

ブート回数45回、通電時間3時間であたりですね。

```sh
$ sudo smartctl --scan
/dev/nvme0 -d nvme # /dev/nvme0, NVMe device

$ sudo smartctl --all /dev/nvme0
smartctl 7.3 2022-02-28 r5338 [aarch64-linux-6.12.47+rpt-rpi-2712] (local build)
Copyright (C) 2002-22, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Number:                       KBG40ZNS256G KIOXIA
Serial Number:                      XXXXXXXXXXXX
Firmware Version:                   AEGA0103
PCI Vendor/Subsystem ID:            0x1e0f
IEEE OUI Identifier:                0x8ce38e
Total NVM Capacity:                 256,060,514,304 [256 GB]
Unallocated NVM Capacity:           0
Controller ID:                      0
NVMe Version:                       1.3
Number of Namespaces:               1
Namespace 1 Size/Capacity:          256,060,514,304 [256 GB]
Namespace 1 Formatted LBA Size:     512
Namespace 1 IEEE EUI-64:            8ce38e 040447d15e
Local Time is:                      Thu Oct  9 16:43:20 2025 JST
Firmware Updates (0x14):            2 Slots, no Reset required
Optional Admin Commands (0x001f):   Security Format Frmw_DL NS_Mngmt Self_Test
Optional NVM Commands (0x005f):     Comp Wr_Unc DS_Mngmt Wr_Zero Sav/Sel_Feat Timestmp
Log Page Attributes (0x0e):         Cmd_Eff_Lg Ext_Get_Lg Telmtry_Lg
Maximum Data Transfer Size:         512 Pages
Warning  Comp. Temp. Threshold:     82 Celsius
Critical Comp. Temp. Threshold:     86 Celsius

Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     3.60W       -        -    0  0  0  0        1       1
 1 +     2.60W       -        -    1  1  1  1        1       1
 2 +     2.20W       -        -    2  2  2  2        1       1
 3 -   0.0500W       -        -    4  4  4  4      800    1200
 4 -   0.0050W       -        -    4  4  4  4     3000   32000

Supported LBA Sizes (NSID 0x1)
Id Fmt  Data  Metadt  Rel_Perf
 0 +     512       0         3
 1 -    4096       0         1

=== START OF SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

SMART/Health Information (NVMe Log 0x02)
Critical Warning:                   0x00
Temperature:                        41 Celsius
Available Spare:                    100%
Available Spare Threshold:          10%
Percentage Used:                    0%
Data Units Read:                    17,952 [9.19 GB]
Data Units Written:                 216,349 [110 GB]
Host Read Commands:                 317,870
Host Write Commands:                542,119
Controller Busy Time:               6
Power Cycles:                       45
Power On Hours:                     3
Unsafe Shutdowns:                   19
Media and Data Integrity Errors:    0
Error Information Log Entries:      0
Warning  Comp. Temperature Time:    0
Critical Comp. Temperature Time:    0
Temperature Sensor 1:               41 Celsius

Error Information (NVMe Log 0x01, 16 of 256 entries)
No Errors Logged
```

# 設定手順

設定は大きく分けて、準備、コピー、設定変更の3つのステップで進めます。

## OSの準備と起動

まず、通常通りmicroSDカードにOSをインストールし、基本的な設定を済ませます。

1. OSの書き込み: Raspberry Pi Imagerを使い、最新のRaspberry Pi OS（64-bit推奨）をmicroSDカードに書き込みます。2025/10/1にリリースされたTrixie (Debian13)を選びました。

2. 初回起動: 作成したmicroSDカードとNVMe SSDをRaspberry Pi 5に接続し、起動します。

3. 初期設定とアップデート: OSの初期設定（ユーザー名、パスワード、Wi-Fi設定など）を完了させ、ターミナルで以下のコマンドを実行してシステムを最新の状態に更新しておきます。

```sh
$ sudo apt update
$ sudo apt upgrade
```

## NVMe SSDへのルートファイルシステムのコピー

次に、microSDカードのルートファイルシステム全体をNVMe SSDにコピーします。

1. NVMe SSDの確認: ターミナルで `lsblk` コマンドを実行し、NVMe SSDが認識されていることを確認します。通常、 `/dev/nvme0n1` のように表示されます。
2. `fdisk` で地道に作業する方法もありますが、私はSD Card Copierを使いました。GUIで簡単にコピーできます。**New Partition UUIDs** にチェックを入れるのを忘れないようにします。SD Cardを選択し、NVMe SSDを選択してコピーを実行します。

![SD Card Copier](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3f85a625-f1b4-4b56-baa8-0260b5ac0e1d.jpeg)

## 起動設定の変更

最後に、起動時にNVMe SSDをルートとしてマウントするように設定を変更します。

### PARTUUIDの確認

NVMe SSDの `PARTUUID` （パーティション固有のID）を調べておきます。このIDを使うことで、デバイス名が変わっても確実にパーティションを指定できます。出力の中から `/dev/nvme0n1p2` の行を探し、 `bbbbbbbb-02` の部分をコピーしておきます。

```sh
$ lsblk -o NAME,PARTUUID
NAME        PARTUUID
loop0
mmcblk0
├─mmcblk0p1 aaaaaaaa-01
└─mmcblk0p2 aaaaaaaa-02
zram0
nvme0n1
├─nvme0n1p1 bbbbbbbb-01
└─nvme0n1p2 bbbbbbbb-02
```

### cmdline.txt を編集

ブートローダーの設定ファイルを編集し、ルートファイルシステムの場所を変更します。

```sh
$ sudo nano /boot/firmware/cmdline.txt
```

ファイル内に `root=PARTUUID=...` という記述があります。この `...` の部分を、先ほどコピーしたNVMe SSDの `PARTUUID` に書き換えます。

変更前：
```
console=serial0,115200 console=tty1 root=PARTUUID=aaaaaaaa-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cfg80211.ieee80211_regdom=JP
```

変更後：
```
console=serial0,115200 console=tty1 root=PARTUUID=bbbbbbbb-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cfg80211.ieee80211_regdom=JP
```

（ `bbbbbbbb-02` はNVMe SSDの `PARTUUID` ）

### NVMeをマウント

```sh
$ sudo mkdir /mnt/ssd1
$ sudo mount /dev/nvme0n1p1 /mnt/ssd1
$ sudo mkdir /mnt/ssd2
$ sudo mount /dev/nvme0n1p2 /mnt/ssd2
```

### fstab の編集

NVMe SSD上の `fstab` ファイルを編集し、起動時のマウント設定が正しくなるようにします。

```sh
$ sudo nano /mnt/ssd2/etc/fstab
```

`PARTUUID` をNVMe SSDのものに書き換えます。

```
proc            /proc           proc    defaults          0       0
PARTUUID=bbbbbbbb-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=bbbbbbbb-02  /               ext4    defaults,noatime  0       1
```

### PCIe Gen3設定

ついでにPCIe Gen3に設定します。

```sh
$ sudo nano /boot/firmware/config.txt
```

allセクションに以下を追加します。

```
[all]
dtparam=pciex1_gen=3
```

`raspi-config` を使ってPCIe Gen3を有効にすることもできます。

```sh
$ sudo raspi-config
```

6 Advanced Options > A9 PCIe Speed > Yes の手順です。`config.txt` に自動的に追加されます。

:::note info
ちなみに、 `dtparam=pciex1` 、 `dtparam=nvme`を有効にする必要はありません。デフォルトになりました。
:::

## 再起動と確認

再起動: すべての設定が完了したら、Raspberry Piを再起動します。

```sh
$ sudo reboot
```

再起動後、以下のコマンドでNVMe SSDが正しくマウントされているか確認します。

```sh
$ df -h
```

出力例：
```sh
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/nvme0n1p2  234G  6.9G  215G   4% /
...
/dev/nvme0n1p1  511M   87M  424M  18% /boot/firmware
...
/dev/mmcblk0p1  510M   87M  424M  17% /media/pi/bootfs
```

上記のように `/` の `Filesystem` が `/dev/nvme0n1p2` になっていれば設定は完了です。

# ブートシーケンス

1. 電源投入
2. Raspberry Pi 5のブートローダーが起動し、microSDカードから起動を試みる（ `/boot/firmware/start*.elf` を読み込む）
3. microSDカードの `/boot/firmware/cmdline.txt` を読み込み、 `root=PARTUUID=bbbbbbbb-02` を確認
4. NVMe `SSDのPARTUUID=bbbbbbbb-02` を持つパーティションをルートファイルシステムとしてマウント
5. OSがNVMe SSD上で起動
6. `fstab` に基づき、 `/boot/firmware` をNVMe SSDの `PARTUUID=bbbbbbbb-01` にマウント

:::note warn
ブート順序はmicroSDカードをNVMeより先にします。さもないと、NVMeブートエラーになってmicroSDカードからブートに移るまでに結構な時間待たされます。
:::

# おわりに

Raspberry Pi 5 M.2 HAT+で単独ブートできないNVMe SSDをルートファイルシステムにする方法を紹介しました。

この方法ですと、microSDカードを指しておかないと起動できないのですが、OSが起動すればもう使いません。抜いてしまっても問題ありません。microSDカードのFAT32パーテーションが、普通の外部ストレージのように自動的にマウントされてしまいます。うっかりいじらないように注意してください。

いまはディスプレイとキーボードを外し、WayVPNとSSHで接続しています。いずれPCIeを[LLM‑8850 Card](https://docs.m5stack.com/ja/ai_hardware/LLM-8850_Card)に換装したいですね。
