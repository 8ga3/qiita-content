---
title: microROS-AgentとrqtをColima上のDocker Containerで動かす(Apple Silicon対応)
tags:
  - Docker
  - ROS2
  - rqt
  - micro-ROS
  - colima
private: false
updated_at: '2025-10-12T00:24:40+09:00'
id: 87b29390a783c07fee19
organization_url_name: access
slide: false
ignorePublish: false
---
# 概要

macOS上でmicroROS-AgentをColima上のDockerで動かす手順を説明します。

# 動作環境

- MacBook Pro (14-inch, 2023.11)
- Apple Silicon M3
- macOS Sequoya 15.7.1
- ROS2 Kilted

# Colimaをインストール

Docker Desktopからその他へ乗り換えた人も多いかと思います。ここではColima導入を簡単に手順を説明します。

```sh
$ brew install colima docker docker-compose docker-buildx
```

動作確認

```sh
$ colima start --arch aarch64 --cpu 8 --memory 16 --disk 100
$ docker version
Client: Docker Engine - Community
 Version:           28.5.1
 API version:       1.47 (downgraded from 1.51)
 Go version:        go1.25.2
 Git commit:        e180ab8ab8
 Built:             Wed Oct  8 02:50:32 2025
 OS/Arch:           darwin/arm64
 Context:           colima

Server: Docker Engine - Community
 Engine:
  Version:          27.4.0
  API version:      1.47 (minimum version 1.24)
  Go version:       go1.22.10
  Git commit:       92a8393
  Built:            Sat Dec  7 10:39:01 2024
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.7.24
  GitCommit:        88bf19b2105c8b17560993bee28a01ddc2f97182
 runc:
  Version:          1.2.2
  GitCommit:        v1.2.2-0-g7cb3632
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

$ docker compose version
Docker Compose version 2.40.0

$ docker buildx version
github.com/docker/buildx v0.29.1 Homebrew

$ colima version
colima version 0.9.1
git commit: 0cbf719f5409ce04b9f0607b681c005d2ff7d94a

runtime: docker
arch: aarch64
client: v28.5.1
server: v27.4.0

$ colima stop
```

サービスとして起動します。ログインまたはブート時に自動実行されます。

```sh
$ brew services start colima
```

サービスをストップします。ログインまたはブート時に自動実行を無効にします。

```sh
$ brew services stop colima
```

サービスとして起動します。

```sh
$ brew services run colima
```

サービスを停止します。ログインまたはブート時に自動実行は変更しません。

```sh
$ brew services kill colima
```

# microROS-AgentとrqtをColima上のDocker Containerで動かす

ROS2 KiltedをインストールしたDockerイメージを使います。`docker-compose.yaml` に２行追加します。

`DISPLAY`は、ホスト側のmacOSのXQuartzで表示するための設定です。

`LIBGL_ALWAYS_INDIRECT`を設定すると、libGL はハードウェア アクセラレーションではなく必ず間接レンダリングを使うようになります。

```diff_yaml:docker-compose.yaml
version: "2.1"

services:
  micro-ros-agent:
    image: microros/micro-ros-agent:kilted
    container_name: micro-ros-agent
    network_mode: host
    stdin_open: true
    tty: true
    environment:
      - ROS_DOMAIN_ID=8
    ports:
      - 8888:8888/udp
    command: udp4 --port 8888 --verbose 6

  rqt:
    image: osrf/ros:kilted-desktop
    container_name: rqt-visualizer
    network_mode: host
    ipc: host
    pid: host
    tty: true
    privileged: true
    device_cgroup_rules:
      - 'c 189:* rmw'
    environment:
      - ROS_DOMAIN_ID=8
+      - DISPLAY=host.docker.internal:0
      - QT_X11_NO_MITSHM=1
+      - LIBGL_ALWAYS_INDIRECT=1
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
    command: bash -c "source /opt/ros/kilted/setup.bash && rqt"
```

# XQuartz

## XQuartzのインストール

```bash
brew install xquartz
open -a XQuartz
```

## XQuartzの設定

- XQuartzのメニューから**XQuartz**→**設定**を選択
- **セキュリティ**タブを開き、**ネットワーク・クライアントからの接続を許可**にチェックを入れる
- XQuartzを再起動

## ターミナルでxhostを実行

コンテナのGUIをホスト側で表示できるように許可します。

```bash
xhost +localhost
```

## XQuartzの設定

初期状態を確認します。

```sh
$ defaults read org.xquartz.x11
{
    "NSWindow Frame x11_apps" = "395 457 454 299 0 0 1800 1125 ";
    "NSWindow Frame x11_prefs" = "1080 195 584 369 0 0 1800 1125 ";
    SUHasLaunchedBefore = 1;
    SULastCheckTime = "2025-09-29 14:30:43 +0000";
    "app_to_run" = "/opt/X11/bin/xterm";
    "cache_fonts" = 1;
    "done_xinit_check" = 1;
    "enable_iglx" = 0;
    "login_shell" = "/bin/sh";
    "no_auth" = 0;
    "nolisten_tcp" = 0;
    "startx_script" = "/opt/X11/bin/startx -- /opt/X11/bin/Xquartz";
}
```

:::note warn
古い情報では`org.macosforge.xquartz.X11`になっていますが、現在は`org.xquartz.X11`です。
:::

xtermを起動しないようにします。（これはお好みで設定してください。）

```sh
defaults write org.xquartz.X11 app_to_run /usr/bin/true
```

Indirect GLX を有効にします。

```sh
defaults write org.xquartz.X11 enable_iglx -bool true
```


# glxgearsをテスト実行

`glxgears`を動かしてみます。

## Dockerfile

```docker
FROM alpine

RUN apk --no-cache add mesa-demos

CMD ["glxgears"]
```

## ビルド

```sh
$ docker build -t glxgears .
```

エラーはでますが動きました。

```sh
$ docker run --rm -e DISPLAY=host.docker.internal:0 -v ~/.Xauthority:/root/.Xauthority glxgears
No matching fbConfigs or visuals found
glx: failed to create drisw screen
```

`LIBGL_ALWAYS_INDIRECT`を設定すると、エラーは出なくなる。

```sh
$ docker run --rm -e DISPLAY=host.docker.internal:0 -e LIBGL_ALWAYS_INDIRECT=1 -v ~/.Xauthority:/root/.Xauthority glxgears
```

![glxgears.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/dc1d5587-f0ac-4bb5-92ad-940815e4cd7b.png)

ソフトウェアでレンダリングするので、パフォーマンスは良くないです。


# microROS-Agentとrqtを実行

`docker-compose.yaml`のあるディレクトリで以下のコマンドを実行します。

```sh
$ brew services run colima
$ docker-compose up
```

とりあえずウィンドウだけです。お見せできるようなデータはありません。申し訳ありません。

![rqt.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ed611624-21f1-4a6d-8c8c-b4539e5117db.png)

# まとめ

Ubuntuマシンか、Parallels DesktopのようなVM上のUbuntuでやるのが一番楽です。でも、いつも使っているmacOS上でサクッと動かしたいのです。Colimaを使うと、Docker Desktopを使わずに済むので、ライセンスの問題もクリアできます。

# 参考

https://qiita.com/Lightning7329/items/e2fd6045782230cebb61

