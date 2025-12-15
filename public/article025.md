---
title: ArduPilot SITLのDockerイメージをダイエット
tags:
  - Docker
  - ardupilot
private: false
updated_at: '2025-12-16T08:13:04+09:00'
id: 384081801628753efb7c
organization_url_name: access
slide: false
ignorePublish: false
---
# はじめに

ドローンとはなにか、もはや説明不要かもしれません。数千円のトイドローンや、撮影用・レース用ドローンは数万円くらいから購入できます。

高度な自立飛行を実現するオープンソースの自律飛行ソフトウェアである[ArduPilot](https://ardupilot.org/)があります。こちらは個人から商用まで幅広く利用されており、ドローンだけでなく、ローバーやボート、潜水艦など多様なプラットフォームに対応しています。

この記事では、ArduPilotのSITL(Software In The Loop)シミュレーションをDockerコンテナ上で動かすための軽量なDockerイメージの作成方法を紹介します。皆様のドローン開発環境の改善に役立てば幸いです。

:::note warning
ドローンの仕組みやArduPilot SITLの基本的な使い方については説明しません。あらかじめご了承ください。
:::

SITLを使えば、羽田空港でフライトなんてことも可能です！

![qgc_drone_three.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5cf5792c-944b-4cbe-a570-f803d3d256df.jpeg)

GitHubリポジトリは[こちら](https://github.com/8ga3/ardupilot_sitl_docker)です。

https://github.com/8ga3/ardupilot_sitl_docker

## ターゲットユーザー

- ArduPilotを使ってドローン開発を行っているエンジニア
- ArduPilot公式Dockerイメージのサイズが気になる方
- SITLだけを手軽に試したい方
- ファームウェアのビルドを必要としない方
- ArduPilotのリリースバージョンを複数そろえておきたい方（古すぎるのおそらくNG。ベースのDockerイメージを変えないといけないかもしれない。）

## 特徴

- とにかく軽量：マルチステージビルドを活用し、不要なビルドツールや依存関係を排除
- シンプルな構成：SITLのみを含む最小限のセットアップ
- セキュリティ：ノンルートユーザーで実行
- マルチインスタンス対応：複数のSITLインスタンスを同時に起動可能
- ArduPilotビルド環境不要：ホストにArduPilotのビルド環境を用意する必要なし

## 利用シーン

- ローカル開発環境でのSITLテスト
- CI/CDパイプラインでの自動テスト
- デモ環境の構築
- 教育目的での利用

---

# 使い方

## ビルドと起動
コンテナはバックグラウンドで起動します。

:::note info
この時点でSITLはまだ起動していません。
:::

```sh
docker compose up -d
```

## ArduPilot SITL 起動

基本的にGitHubをクローンした後の手順と同じです。`sim_vehicle.py`を実行する際に、`docker exec -it ardupilot-sitl`を付与してコンテナ内で実行します。

ポイント
- `--no-rebuild`オプションを付与して、ビルドをスキップします。
- `--mavproxy-args`オプションで、MAVProxyの引数を指定します。`--out udp:host.docker.internal:14550`のように、`host.docker.internal`を使用してホスト側へMAVLinkメッセージを送信します。

```sh
docker exec -it ardupilot-sitl \
  ./Tools/autotest/sim_vehicle.py \
  --vehicle ArduCopter \
  --frame X \
  --custom-location 35.54863,139.78096,4.2,45 \
  --no-rebuild \
  --wipe-eeprom \
  --mavproxy-args="--out udp:host.docker.internal:14550 --state-basedir=/tmp/mavlink-sitl"
```

## 複数のインスタンスのArduPilot SITLを起動

- システムID (`--sysid`) とインスタンス番号 (`--instance`) をそれぞれ変更して起動します。
- `--mavproxy-args`オプションで、MAVLinkメッセージの送信先ポート(`--out`)と状態保存ディレクトリ(`--state-basedir`)を変更します。
- QGroundControlで2つ目以降のデフォルト値以外のインスタンスに接続する場合、アプリケーション設定の「通信リンク」に、対応するUDPポートを追加してください。

以下、2つのインスタンスを起動する例です。

1つ目のシェル
```sh
docker exec -it ardupilot-sitl \
  ./Tools/autotest/sim_vehicle.py \
  --vehicle ArduCopter \
  --frame X \
  --custom-location 35.54863,139.78166,4.2,45 \
  --no-rebuild \
  --wipe-eeprom \
  --sysid=1 \
  --instance=0 \
  --mavproxy-args="--out udp:host.docker.internal:14550 --state-basedir=/tmp/mavlink-sitl-1"
```

2つ目のシェル
QGroundControlで接続する場合、UDPポート14560を指定してください。
```sh
docker exec -it ardupilot-sitl \
  ./Tools/autotest/sim_vehicle.py \
  --vehicle ArduCopter \
  --frame X \
  --custom-location 35.54821,139.78196,4.2,45 \
  --no-rebuild \
  --wipe-eeprom \
  --sysid=2 \
  --instance=1 \
  --mavproxy-args="--out udp:host.docker.internal:14560 --state-basedir=/tmp/mavlink-sitl-2"
```

---

# 仕組み

## Dockerfile解説

マルチステージビルドを活用し、ビルドステージとランタイムステージに分けています。なるべくレイヤー数を減らすために、各ステージでの`RUN`コマンドはできるだけまとめています。ビルドに必要なツールや依存関係はビルドステージにのみ含め、最終的なイメージにはSITLの実行に必要な最小限のファイルだけをコピーしています。

### ビルドステージ

python3.13-bookworm-slimベースのイメージを使用し、ArduPilotのビルドに必要なツールとライブラリをインストールします。UVパッケージマネージャーを利用してPython環境をセットアップし、ArduPilotのソースコードをクローンしてSITLバイナリをビルドします。

バイナリファイルは、Copter、Rover、Plane、Subの4種類をビルドしています。

```Dockerfile
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS builder
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y \
	build-essential \
	ccache \
	g++ \
	gawk \
	git \
	make \
	wget \
	valgrind \
	screen \
	procps \
	libtool \
	libxml2-dev \
	libxslt1-dev \
	python3-dev \
	python3-pip \
	python3-setuptools \
	python3-numpy \
	python3-pyparsing \
	python3-psutil \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Set up Python virtual environment with UV
COPY uv.lock pyproject.toml ./

ENV UV_SYSTEM_PYTHON=true \
    UV_COMPILE_BYTECODE=1 \
    UV_CACHE_DIR=/app/.cache/uv \
    UV_LINK_MODE=copy \
	UV_PYTHON_DOWNLOADS=never

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

ENV PATH="/app/.venv/bin:$PATH"

# Clone Ardupilot repository
ENV ARDUPILOT_VERSION=ArduPilot-4.6
RUN git clone --branch ${ARDUPILOT_VERSION} --depth 1 --recurse-submodules https://github.com/ArduPilot/ardupilot.git /app/ardupilot
WORKDIR /app/ardupilot

# Build Ardupilot
RUN ./waf distclean
RUN ./waf configure --board sitl
RUN ./waf copter
RUN ./waf rover
RUN ./waf plane
RUN ./waf sub

# Clean up unnecessary packages from virtual environment
WORKDIR /app
RUN uv remove empy packaging
```

### ランタイムステージ

ここでは、python3.13-slim-bookwormベースのイメージを使用し、ビルドステージで作成したSITLバイナリと必要なPythonパッケージをコピーします。よってUVパッケージマネージャーは不要です。セキュリティ向上のためにノンルートユーザー(`nonroot`)を作成し、そのユーザーでコンテナを実行するように設定しています。

`sim_vehicle.py`スクリプトは、ソースコードがあるフォルダがないとエラーで止まってしまうため、必要最低限のフォルダを作成しておきます。ソースコード自体は必要ありません。

実行バイナリ本体は、`arducopter`・`ardurover`・`arduplane`・`ardusub`の4つをコピーしています。

最後`ENTRYPOINT`でコンテナ起動時に無限ループで待機するようにしています。

```Dockerfile
FROM python:3.13-slim-bookworm
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies
RUN apt-get update && apt-get install -y procps \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd --system --gid 999 nonroot \
 && useradd --system --gid 999 --uid 999 --create-home nonroot

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"

# Prepare Ardupilot directories
RUN mkdir -p /app/build/sitl \
	/app/Tools/autotest \
	/app/AntennaTracker \
	/app/ArduCopter \
	/app/ArduPlane \
	/app/ArduSub \
	/app/Rover

COPY --from=builder /app/ardupilot/build/sitl/bin /app/build/sitl/bin
COPY --from=builder /app/ardupilot/Tools/autotest/sim_vehicle.py /app/Tools/autotest/sim_vehicle.py
COPY --from=builder /app/ardupilot/Tools/autotest/run_in_terminal_window.sh /app/Tools/autotest/run_in_terminal_window.sh
COPY --from=builder /app/ardupilot/Tools/autotest/default_params /app/Tools/autotest/default_params
COPY --from=builder /app/ardupilot/Tools/autotest/pysim /app/Tools/autotest/pysim

RUN chown nonroot:nonroot /app

# Switch to non-root user
USER nonroot

ENTRYPOINT [ "tail", "-F", "/dev/null" ]
```

## compose.yaml解説

ARDUPILOT_VERSIONビルド引数を指定して、使用するArduPilotのバージョンを変更できるようにしています。

tmpfsを使用して、SITLのログファイルや地形データをコンテナの一時ファイルシステム上に保存するようにしています。これにより、ホスト側のストレージを消費せず、パフォーマンスも向上します。ユーザーIDとグループIDを999に指定して、ノンルートユーザーでのアクセス権限を確保しています。

entrypointを`tail -F /dev/null`に設定して、コンテナが起動し続けるようにしています。ここを書き換えてSITLを直接起動することも可能です。

```yaml
services:
  ardupilot-sitl:
    container_name: ardupilot-sitl
    image: ardupilot-sitl
    pull_policy: build
    build:
      context: .
      dockerfile: Dockerfile
      args:
        # https://github.com/ArduPilot/ardupilot のブランチ名
        - ARDUPILOT_VERSION=ArduPilot-4.6
    tmpfs:
      - /app/logs:uid=999,gid=999
      - /app/terrain:uid=999,gid=999
    # containerを実行し続ける
    entrypoint: "tail -F /dev/null"
```

---

# 結果

ArduPilot公式の[ardupilot/ardupilot-dev-base](https://hub.docker.com/r/ardupilot/ardupilot-dev-base)をpullして、必要なパッケージをインストールしSITLをビルドすると、イメージサイズは約6GB超になっていた気がします。イメージファイルはamd64環境のみで、arm64環境は用意されていません。

マルチステージビルドを使ってない[シンプルなDockerfile](https://github.com/8ga3/ardupilot_sitl_docker/tree/main/simple)でイメージファイルを作成すると、約4GBになりました。（Apple Silicon M3 macOS 15.7.2環境でビルド・確認）

マルチステージビルドで作成した今回のDockerイメージを環境を変えて比較してみました。

## MacBook Pro (14-inch, Nov 2023)
- プロセッサ名:	Apple M3 Pro
- メモリ:	36 GB
- システムのバージョン:	macOS 15.7.2
- colima version 0.9.1
  - runtime: docker
  - client: v29.1.2
  - server: v27.4.0
  - Architecture: aarch64

## MacBook Pro (15-inch, 2019)
- プロセッサ名:	6コアIntel Core i7 2.2 GHz
- メモリ:	16 GB
- システムのバージョン:	macOS 15.7.2

- Docker Desktop
  - Version 4.54.0 (212467)
  - Engine: 29.1.2
  - Compose: v2.40.3-desktop.1
  - Architecture: amd64

## Raspberry Pi 5
- プロセッサ名: Broadcom BCM2712, Cortex-A76 (ARM v8) 64-bit SoC @ 2.0GHz
- メモリ: 8GB LPDDR4-3200 SDRAM
- ストレージ: NVMe SSD
- OS: Raspberry Pi OS 64-bit (Trixie, Debian 13ベース)

- Docker Engine
  - Client Version: 26.1.5+dfsg1
  - Server Version: 26.1.5+dfsg1
  - Architecture: aarch64

## 比較結果

| 環境                      | パッケージ     | イメージサイズ | ビルド時間 |
|---------------------------|----------------|----------------|------------|
| MacBook Pro  Apple M3 Pro | Colima         | 621MB          | 566.9s     |
| MacBook Pro  Intel i7     | Docker Desktop | 443MB          | 535.3s     |
| Raspberry Pi 5            | Docker Engine  | 464MB          | 795.2s     |

**約400〜600MB台**と非常に軽量です。素のDockerfileで作成した場合と比べて、**約1/8〜1/10程度**のサイズに削減できました。イメージファイルをセーブして他のマシンにコピーする場合も楽ですね。

X86_64環境の方がARM64環境よりもイメージサイズが小さくなるのは普通のことですが、Apple M3 Pro環境が大きくなりすぎていて謎です。ビルド時間もIntelが一番速いです。コンテナが使用しているCPUコア数、メモリ量、ストレージなど条件がバラバラなので、あくまで参考値として見てください。

:::note info
MacBook Pro (14-inch, 2023.11)のホスト上でArduPilot SITL4種をビルドすると、かかった時間は合計で約1分30秒台でした。
:::

---

# おわりに

恥ずかしながらDockerをなんとなく雰囲気で使っていました。近年の流行に乗っかって、ようやくちゃんと勉強し始めた次第です。マルチステージビルドを活用することで、不要なファイルやパッケージを排除し、非常に軽量なDockerイメージを作成できることがわかりました。個人的には大満足の結果です。

🎄🎂🎁⛄
