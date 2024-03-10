---
title: Ubuntu 22.04をLenovo IdeaPad MIIX 320-10ICRにインストール
tags:
  - 'IdeaPad'
  - 'MIIX'
  - 'Ubuntu'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# 概要

Lenovo IdeaPad MIIX 320-10ICRを、これまで普通にWindowsPCとして使用していました。CPUがAtom x5-Z8350ということもありWindows11にアップグレードできないですし、そもそも低スペックすぎて辛いです。

放置していてもなんなので、Ubuntu 22.04を入れておこうと思いました。

# スペック

* CPU: Atom x5-Z8350 (1.44/1.92GHz)
* GPU: Intel HD Graphics 400
* Mem: DDR3L-1600 SDRAM 4GB
* Strage: 64GB eMMC

# 準備

Ubuntu 22.04のISOファイルをダウンロードしておきます。
別のUbuntuマシンでUSBメモリに書き込んでおきます。USB 3.0のメモリを使いましたが、USB 2.0のポートにさして作業しました。USB 3.0ポートならインストールは速かったかもしれません。

# Miix設定

もうこのマシンでWindows使わないし、ストレージは64GBしかないので、デュアルブートにはしませんでした。一応、Windowsで回復ドライブを作っておきました。

BIOSの設定をしておきます。電源を入れたらFn + F2を押します。**Configuration** 画面に切り替え、**Intel Pratform Trust Technology** と **Secure Boot** をDisableにします。Saveして再起動したら、Fn + F12を押して起動メニューからUbuntuインストーラーを選択できます。

![miix_bios.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/bf74efee-7e67-0a22-dd16-97a4c4e301b8.jpeg)

# Ubuntu 22.04 インストール

![miix_ubuntu.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/81101365-5a75-2c8a-0306-179180b1905e.jpeg)

このとおり何事もなくインストール完了しました。画面がHiDPIになっています。このままだと画面が狭いので、**設定** の **ディスプレイ** で **任意倍率のスケーリング** をオフにします。画面サイズ200%だったのが100%になり画面が広がりました。ちょっと小さくて見づらいですね。

# インストール後

Chromium、Visual Studio Code、GIMP、Inkscape、FreeCADを追加してみました。問題なく動いているようです。Windowsよりずいぶん快適です。
