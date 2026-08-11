---
title: Babylon.js 9.0 の新機能 Geospatial で日本の地形を3D化するモジュール jpmap_terrain 紹介
tags:
  - jpmap_terrain
  - TypeScript
  - Babylon.js
  - Geospatial
  - 地理院タイル
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

ブラウザで動く3D地図アプリやゲームを作ってみたい。

でも、地図データを用意したり、サーバーを運用したりすると、どうしてもコストが気になります。

そこで、できるだけランニングコストをかけずに、日本の実際の地形を使った3Dコンテンツを作るプロジェクトを始めました。

それが [jpmap_terrain](https://github.com/8ga3/jpmap_terrain) です。

https://github.com/8ga3/jpmap_terrain

# まずはデモをご覧ください

実際の動作はこちらから確認できます。

**デモサイト**

https://jpmap-terrain.netlify.app/

3D地形ビューアを中心に、次のようなデモを用意しています。

- 航空写真表示

![航空写真表示](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/8ec0829b-9d5c-40f2-805c-5c3ba2c18c3f.jpeg)

- タイムラプス

![タイムラプス](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/12caa78d-cb2b-4461-a5f2-afb1fb25a8aa.jpeg)

- 距離計測

![距離計測](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/84e5f75f-563e-4f2b-a274-baba8afc06a5.jpeg)

- GPXビューア

![GPXビューア](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/06d86128-edd9-458e-9f69-b055b4f7d34b.jpeg)

- アバター配置・操作

![アバター配置・操作](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5b054662-8552-4b6f-9359-f962c5e95fca.jpeg)

- Boidsシミュレーション

![Boidsシミュレーション](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/09023e84-099b-48f0-871f-f5122224fab0.jpeg)

- フライトデモ

![フライトデモ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/cde20ae1-ae8b-4de6-913a-ecb1a156f973.jpeg)

- Artillery Game

![Artillery Game](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/0d9fe59d-8ac6-4c5e-ab75-870f532f9407.jpeg)

- WebXR 箱庭ジオラマ

![WebXR 箱庭ジオラマ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/cc6c17b9-7b94-46b2-a88d-2a510236e2de.jpeg)

いろいろ試せるようにしているので、気になるものから触ってみてください。

:::note warn
デモサイトはNetlifyの無料枠で運用しています。無料枠のクレジットを使い切った場合など、一時的にページを表示できないことがあります。あらかじめご了承ください。
:::

ソースコードはこちらで公開しています。

**GitHub**

https://github.com/8ga3/jpmap_terrain

# `jpmap_terrain`とは

`jpmap_terrain`は、国土地理院の地理院タイルを使って、ブラウザ上に3D地形を表示するプロジェクトです。

標高タイルから地形の高さを読み取り、Babylon.jsのメッシュを生成しています。

さらに、標準地図や航空写真のタイルをテクスチャとして貼り付けることで、見慣れた日本の地形をそのまま3D空間に持ってくることができます。

目指しているのは、単なる3D地図ビューアではありません。

地形をそのままゲームの舞台にしたり、シミュレーションや教育コンテンツに使ったりできるようにすることです。

# ランニングコストを抑えた構成

地形データを自分でサーバーに保存して配信するのではなく、ブラウザから必要な地理院タイルを取得する構成にしています。

そのため、アプリケーション側で大規模な地形データベースや専用サーバーを用意しなくても動かせます。

静的ホスティングと組み合わせれば、比較的低コストで公開できます。

もちろん、地理院タイルの利用規約や出典表示のルールには従う必要があります。

# Babylon.js 9.0 のGeospatialを使っています

このプロジェクトでは、Babylon.js 9.0 のGeospatial関連機能を利用しています。

緯度・経度を地球規模の3D空間へ変換し、GeospatialCameraやECEF座標系を使って地形を表示します。

地球規模の座標を扱うと、カメラ周辺で浮動小数点誤差が問題になることがあります。

そこで、カメラ付近を原点として扱うfloating originも利用しています。

これにより、広い範囲を表示しながら、カメラの近くではできるだけ安定した描画ができるようにしています。

```typescript
import { JpmapTerrain } from "jpmap-terrain";

const container = document.getElementById("terrain-viewer");

if (!container) {
  throw new Error("Terrain container was not found.");
}

const viewer = await JpmapTerrain.create(container, {
  engine: "webgpu",
  lat: 35.681236,
  lon: 139.767125,
  altitude: 2000,
  azimuth: 0,
  tilt: 45,
  mapType: "standard",
});
```

WebGPUに対応していない環境では、WebGL2へ切り替えることもできます。

```typescript
const viewer = await JpmapTerrain.create(container, {
  engine: "webgl2",
  lat: 35.3606,
  lon: 138.7274,
  altitude: 8000,
  azimuth: 0,
  tilt: 45,
  mapType: "photo",
});
```

# 地形をゲームの舞台にする

地形上のクリック位置や標高を取得できるので、地面に沿ってオブジェクトを移動させることもできます。

現在は、次のようなデモを作っています。

- 地形上を歩くアバター
- 飛行機のフライトデモ
- GPXルートの表示
- 3Dモデルの配置
- 地形を利用した砲撃ゲーム
- Boidsによる群衆シミュレーション

例えば、富士山へカメラを移動させる処理は次のように書けます。

```typescript
await viewer.flyTo({
  lat: 35.3606,
  lon: 138.7274,
  altitude: 8000,
  duration: 2000,
});
```

実在する場所をそのままゲームやシミュレーションの舞台にできるのが、このプロジェクトの面白いところです。

# 箱庭ジオラマとWebXR

`JpmapDiorama`では、実際の地形を手元サイズの箱庭として表示できます。

現在は富士山周辺を初期地点として、地形をテーブル上のジオラマのように表示しています。

```typescript
import { JpmapDiorama } from "jpmap-terrain";

const mount = document.getElementById("diorama");

if (!mount) {
  throw new Error("Diorama container was not found.");
}

const diorama = await JpmapDiorama.create(mount, {
  center: {
    lat: 35.3436,
    lon: 138.7203,
  },
  footprintHalfSizeM: 800,
  tableRadiusM: 0.35,
});
```

箱庭では、次のような操作ができます。

- 地図の移動
- 拡大・縮小
- 箱庭の回転
- 高さの変更
- 標準地図と航空写真の切り替え
- XRコントローラー操作
- タッチHUD操作

対応デバイスでは、WebXRを使って実空間上に地形の箱庭を表示できます。

WebXRを実機で動かすためのHTTPSやセキュアコンテキストの準備については、別の記事で詳しく紹介する予定です。

# なぜ日本国内のみなのか

`jpmap_terrain`は地理院タイルの標高データを利用しているため、基本的に日本国内が対象です。

詳細な標高データは主に日本周辺で提供されているため、世界中の地形を自由に表示できるサービスではありません。

ただ、日本国内の地形を扱うなら、かなり面白いことができます。

- 富士山や山岳地帯を3D表示する
- 実際の道路や地形を使ったゲームを作る
- 防災や地形学習に利用する
- GPXや飛行ルートを立体的に確認する
- 日本全国を舞台にしたシミュレーションを作る

「実在する場所をゲームの舞台にしたい」という用途とは、相性がよいと思います。

# セットアップ

```shell
git clone https://github.com/8ga3/jpmap_terrain.git
cd jpmap_terrain

npm install
npm start
```

ブラウザで次のURLを開きます。

```text
http://localhost:8080
```

# まとめ

`jpmap_terrain`は、地理院タイルの標高データとBabylon.js 9.0 のGeospatial機能を組み合わせた、国内向けの3D地形プロジェクトです。

ブラウザから必要なタイルを取得する構成にすることで、アプリケーション側のランニングコストを抑えています。

3D地図ビューアとしてだけでなく、ゲーム、シミュレーション、WebXR、教育、防災コンテンツなど、いろいろな方向へ発展させられそうです。

まずはデモサイトで、日本の地形がブラウザ上でどのように表示されるか試してみてください。

- **デモサイト:** https://jpmap-terrain.netlify.app/
- **GitHub:** https://github.com/8ga3/jpmap_terrain
- **Babylon.js:** https://www.babylonjs.com/
- **地理院タイル:** https://maps.gsi.go.jp/development/ichiran.html

:::note warn
地理院タイルを利用する場合は、国土地理院の利用規約および出典表示のルールを必ず確認してください。
:::
