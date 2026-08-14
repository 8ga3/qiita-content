---
title: >-
  Babylon.js 9 の Geospatial（float64）上で float32 の Havok 物理を動かす —
  大砲ゲームで実装した「ステージフレーム」方式
tags:
  - jpmap_terrain
  - TypeScript
  - Babylon.js
  - Geospatial
  - Havok
private: false
updated_at: '2026-08-14T09:08:18+09:00'
id: 279ae178f01151a391a1
organization_url_name: access
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

jpmap_terrainについては、こちらの記事で紹介しています。

https://qiita.com/8ga3/items/33c1b135fcebd3ec8107

地理院タイルの標高データから 3D 地形を作る個人プロジェクト [jpmap_terrain](https://github.com/8ga3/jpmap_terrain) で、デモのひとつとして **Artillery Game（大砲で撃ち合うターン制ゲーム）** を作りました。

砲弾の重力落下・地形へのバウンドは Havok 物理エンジンに任せています。「実際の地形の上に砲弾が落ちて斜面で跳ねる」という絵は、物理エンジンにやらせれば簡単……のはずでした。

![Artillery Game のプレイ画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/5e59fb44-0850-4907-85cd-48e3a7192149.jpeg)

ところがこのデモを Babylon.js 9 の **Geospatial（ECEF 楕円体 + floating origin）** 上に載せようとしたところ、そのままでは Havok が使えないという壁にぶつかりました。

**Geospatial は座標を float64（倍精度）前提で扱うのに対して、Havok は float32（単精度）ベース**だからです。

この記事では、この精度のミスマッチをどう回避して物理シミュレーションを成立させたか、実装コードを追いながら解説します。

- デモのソース: [`src/demos/artillery/`](https://github.com/8ga3/jpmap_terrain/tree/main/src/demos/artillery)
- 環境: `@babylonjs/core` 9.20 / `@babylonjs/havok` 1.3

---

# 問題: ECEF 座標は float32 に載らない

## Geospatial とは

Babylon.js 9 で追加された Geospatial 系の機能は、**地球規模のシーンを実寸で扱う**ためのものです。地形メッシュは ECEF（Earth-Centered, Earth-Fixed）座標に配置されます。ECEF は地球の中心を原点とする直交座標系で、地表の点はおおよそ **6.4 × 10⁶ m** のオーダーになります。

日本のどこかの地表を ECEF で表すと、こんな値です。

```
x = -3,957,000 m
y =  3,310,000 m
z =  3,737,000 m
```

## float32 では 0.76m 刻みになる

ここで float32 の精度を確認します。float32 の仮数部は 23bit なので、値 `x` の刻み幅（ULP）は `2^(e-23)`（`2^e ≤ x < 2^(e+1)`）です。`6.4 × 10⁶` は `2²²`（≈ 4.19e6）と `2²³`（≈ 8.39e6）の間にあるので、

```
ULP = 2^(22-23) = 0.5 [m]
```

つまり **ECEF の大きさを float32 で表すと、表現できる刻み幅は 0.5m** になります。値がもう一段大きい領域（8.39e6 超）に入れば 1m、さらに演算・差分を重ねれば誤差はこれ以上に膨らみます。Babylon 側の実装コメントでも、floating origin リベース時の量子化を **約 0.76m** として記録しています。

砲弾の直径が 20m 前後、地形の起伏の細部が数 m というスケールの中で、座標の刻みが 0.5m 以上もあったら何が起きるか。

- 砲弾の位置が 0.5m 単位でカクカク飛ぶ（ジッター）
- 地形コリジョンメッシュの頂点が量子化されて、なめらかな斜面がガタガタになる
- 衝突法線がブレて、バウンドの方向が毎回変わる
- そもそも砲弾が地面をすり抜ける

**描画側**は Babylon が解決策を用意しています。

| 対策 | 設定箇所 | 役割 |
|---|---|---|
| floating origin | `new Scene(engine, { useFloatingOrigin: true })` | カメラ近傍を原点としてシーンをリベースし、GPU に渡す座標を小さくする |
| high precision matrix | `new Engine(canvas, true, { useHighPrecisionMatrix: true })` | Babylon が追跡する全行列を float64 に切り替える |

この 2 つは**両方セット**でないと意味がありません。実装では次のようになっています。

```ts
// src/scenes/globe.ts
// Large World Rendering: 真の ECEF（百万 m オーダー）でも精度を保つため floating origin を有効化。
// これだけでは不十分で、行列を float64 にする high precision matrix を **engine 側**で
// 有効化する必要がある（engineFactory が globe シーン生成時に useHighPrecisionMatrix を渡す）。
// 両方揃って初めてジッターのない large world になる。
const scene = new Scene(engine, { useFloatingOrigin: true });
scene.useRightHandedSystem = true;
```

`useHighPrecisionMatrix` は Babylon 内部で `PerformanceConfigurator.SetMatrixPrecision()` を呼び、以降生成される `Matrix` のバッキングストアを `Float64Array` に切り替えます。ここが「Geospatial は float64」の実体です。

:::note warn
`useHighPrecisionMatrix` は **最初に生成された Engine** で有効化する必要があり、しかも行列精度は `PerformanceConfigurator` の**グローバル状態**です。PIP（Picture-in-Picture）用のセカンダリ Viewer など Engine を後からもう1つ作ると、そこで `SetMatrixPrecision(false)` が呼ばれて float32 に巻き戻ります。

本プロジェクトでは、一度 high precision を要求したらラッチして以後も維持するようにしています。

```ts
// src/lib/internal/engineFactory.ts
let highPrecisionMatrixLatched = false;

export async function createBabylonEngine(canvas, preferred, options?) {
    if (options?.highPrecisionMatrix) {
        highPrecisionMatrixLatched = true;
    }
    const useHighPrecisionMatrix = highPrecisionMatrixLatched;
    // ...
}
```
:::

## しかし Havok は float32

問題はここからです。Havok は WebAssembly にコンパイルされた C++ の物理エンジンで、**内部の剛体位置・速度・衝突形状はすべて float32** です。Babylon 側の行列を float64 にしても、Havok の WASM ヒープに渡した瞬間に float32 へ落ちます。

つまり、**「砲弾の `PhysicsBody` の位置を ECEF そのままで渡す」ことは原理的にできません。**

`useHighPrecisionMatrix` は描画パイプラインを救ってくれますが、物理エンジンの内部表現までは救ってくれないのです。

---

# 解決策: ステージフレーム（stageRoot）方式

## 発想

ゲームの舞台（戦場）は箱根・芦ノ湖周辺の **数 km 四方**しかありません。地球全体で物理を解く必要はまったくないのです。

そこで採った方針がこれです。

> **物理はローカル ENU 座標（原点近傍の小さい数値）で解き、描画のみ ECEF へ写像する。**

ENU は East-North-Up、ある測地原点に張った局所直交座標系です。ステージ中心を ENU 原点にすれば、砲弾も地形コリジョンも **±3000m 程度の座標**に収まります。この範囲なら float32 の刻みは 0.0002m 程度で、まったく問題になりません。

```
ECEF 絶対座標      : 6,400,000 m オーダー → float32 の刻み 0.5m 以上 ❌
ENU ローカル座標   :     ±3,000 m         → float32 の刻み 約 0.0002m ✅
```

## ENU フレームの構築

まず、測地原点に張る ENU 基底ベクトルを ECEF 空間で求めます。球面測地の標準式です。

```ts
// src/terrain/geo/enu.ts
export const buildEnuFrame = (latDeg, lonDeg, altMeters = 0): EnuFrame => {
    const lat = latDeg * DEG2RAD;
    const lon = lonDeg * DEG2RAD;
    const sinLat = Math.sin(lat), cosLat = Math.cos(lat);
    const sinLon = Math.sin(lon), cosLon = Math.cos(lon);

    const east  = new Vector3(-sinLon, cosLon, 0);
    const north = new Vector3(-sinLat * cosLon, -sinLat * sinLon, cosLat);
    const up    = new Vector3( cosLat * cosLon,  cosLat * sinLon, sinLat);

    return { originEcef: geodeticToEcef(latDeg, lonDeg, altMeters), east, up, north };
};
```

軸割り当ては artillery の物理規約に合わせて **X = East / Y = Up / Z = North** にしています。「Y が上」であることが重要で、これによって重力・仰角・砲身の向きといったゲームロジックを、ごく普通の平面ゲームと同じ感覚で書けます。

この基底を並べれば、そのまま「ENU ローカル → ECEF」のワールド変換行列になります。

```ts
export const buildEnuWorldMatrix = (frame: EnuFrame): Matrix =>
    Matrix.FromValues(
        frame.east.x,       frame.east.y,       frame.east.z,       0,
        frame.up.x,         frame.up.y,         frame.up.z,         0,
        frame.north.x,      frame.north.y,      frame.north.z,      0,
        frame.originEcef.x, frame.originEcef.y, frame.originEcef.z, 1,
    );
```

## stageRoot に全部ぶら下げる

あとはこの行列を持った `TransformNode`（= `stageRoot`）を1つ作り、**ステージの全メッシュをその子にする**だけです。

:::note info
**`TransformNode` とは**

Babylon.js の `TransformNode` は、 **位置・回転・スケールだけを持ち、ジオメトリ（頂点）とマテリアルを持たないノード** です。「描画されない `Mesh`」と考えると分かりやすいと思います。

```
Mesh          = 変換 + ジオメトリ + マテリアル  → 画面に描画される
TransformNode = 変換のみ                        → 画面には何も出ない
```

Babylon.js のシーンはノードの親子ツリーになっていて、子ノードのワールド変換は「自分のローカル変換 × 親のワールド変換」で決まります。したがって `TransformNode` を親に置くと、 **その子孫すべての座標系をまとめて動かす「取っ手」** になります。Unity の空の GameObject、Three.js の `Object3D` / `Group` に相当するものです。

今回は、この性質をそのまま使っています。

```
stageRoot (TransformNode) ── world = ENU→ECEF 行列
├── 赤の大砲       position = (-750, 120, 0)   ← ENU ローカル。人間に分かる数値
├── 青の大砲       position = ( 750, 135, 0)
├── 砲弾 × N       position = (   0,  50, 0)
└── 地形コライダー  6km 四方のグリッド
```

子は `(-750, 120, 0)` のような**小さくて意味の分かる数値**のまま扱えて、Babylon がワールド変換を合成する段階で自動的に ECEF（6.4e6 m オーダー）へ写像してくれる、という仕組みです。ゲームロジック側は ECEF の存在を知らないまま書けます。
:::

## 見えないステージフレームを可視化してみる

`stageRoot` は `TransformNode` なので画面には何も描画されません。地形コライダーも `isVisible = false` です。つまり**このデモの座標基盤は、通常プレイ中は一切目に見えない**わけです。

説明のために、デバッグ用のオーバーレイを被せて撮影してみました。

まずは通常の表示です。箱根・芦ノ湖周辺の地形に赤と青の大砲が置かれているだけに見えます。

![通常表示](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/eaa89aad-510d-4f47-bfeb-ed99b6700920.jpeg)

ここに、`stageRoot` の子としてワイヤーフレームを追加したものがこちらです。

![ステージフレームのオーバーレイ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b7386f67-d8c0-4fed-aa26-d3a773f4f9e8.jpeg)

| 色 | 内容 |
|---|---|
| 🔴 赤い軸 | ENU の **+X = East（東）** |
| 🟢 緑の軸 | ENU の **+Y = Up（上）** = 重力の反対方向 |
| 🔵 青の軸 | ENU の **+Z = North（北）** |
| 🟦 水色のグリッド | ENU 原点の**接平面（y = 0、ステージ原点の海抜0m）**。6km 四方 |
| 🟨 黄色のグリッド | **地形コリジョンメッシュ**（`artillery-collider`）。砲弾が実際に衝突する不可視の物理形状 |

3軸の交点がステージ原点（`stageRoot` のローカル `(0,0,0)`）です。緑の Up 軸は ECEF で見れば**地球の中心から外向きのベクトル**ですが、`stageRoot` の子として `(0, 1800, 0)` を指定するだけで描けています。これがまさに「ローカル ENU で書けば ECEF に写像される」ということです。

水色の平面が地形に対して**傾いて**見えるのも重要なポイントです。この平面はステージ原点における楕円体の接平面なので、水平とは限りません。緯度経度が変われば ECEF 空間での向きも変わります。

そして黄色のグリッドが、砲弾が跳ねる**実体としての地形**です。芦ノ湖の湖面から山頂まで、可視の地形メッシュに沿って波打っているのが分かります。

深度テストを効かせて撮ると、コライダーが可視地形にぴったり張り付いていることがより分かりやすくなります。

![地形コライダーのワイヤーフレーム](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/271966a9-f846-4e24-a420-f14325ad59f5.jpeg)

:::note info
**この可視化のやり方**

一時的なデバッグモジュールを1つ作り、開発サーバー経由で動的 import して流し込んでいます。ポイントは、オーバーレイ用のメッシュも**そのまま `stageRoot` の子にする**だけでよい、ということです。

```ts
const scene = window.viewer.__debugScene;
const root = scene.getTransformNodeByName("artillery-stage-root")!;

// ENU 接平面のグリッド（ローカル座標で素直に書ける）
const grid: Vector3[][] = [];
for (let v = -3000; v <= 3000; v += 500) {
    grid.push([new Vector3(-3000, 0, v), new Vector3(3000, 0, v)]);
    grid.push([new Vector3(v, 0, -3000), new Vector3(v, 0, 3000)]);
}
const plane = CreateLineSystem("dbg-enu-plane", { lines: grid }, scene);
plane.color = new Color3(0.2, 0.9, 1.0);
plane.parent = root;              // ← これだけで ECEF に写像される
plane.renderingGroupId = 1;       // 地形に隠れず常に手前へ

// 不可視のコライダーメッシュを間引いてワイヤー化
const positions = scene.getMeshByName("artillery-collider")!
    .getVerticesData(VertexBuffer.PositionKind)!;
```

`CreateLineSystem` を使う場合、`@babylonjs/core` を個別 import している構成では
`import "@babylonjs/core/Shaders/color.fragment"` / `color.vertex` の副作用 import を
忘れるとシェーダのコンパイルに失敗します（筆者はこれで最初にハマりました）。
:::

```ts
// src/demos/artillery/stageFrame.ts
export const createStageFrame = (scene: Scene, origin: StageOrigin): StageFrame => {
    const frame = buildEnuFrame(origin.lat, origin.lon, origin.alt ?? 0);
    const world = buildEnuWorldMatrix(frame);
    const inv = Matrix.Invert(world);

    const root = new TransformNode("artillery-stage-root", scene);
    // 左手系で det=-1 になりうるため decompose を避け world を直接固定する。
    root.freezeWorldMatrix(world);

    // 重力: ステージローカルの -Y(= -Up)。Havok はワールド重力で解くため up*(-|g|)。
    const gravity = frame.up.scale(DEMO_GRAVITY_Y);
    const downWorld = frame.up.scale(-1);

    return {
        root,
        gravity,
        attach: (node) => { node.parent = root; },
        localToWorld: (local, ref) => Vector3.TransformCoordinatesToRef(local, world, ref),
        worldToLocal: (w, ref)     => Vector3.TransformCoordinatesToRef(w, inv, ref),
        worldDirToLocal: (dir, ref) => Vector3.TransformNormalToRef(dir, inv, ref),
        downWorld,
    };
};
```

`stageRoot` の子はローカル座標（= ENU）で扱えるので、大砲の配置・発射・命中判定といった既存のゲームロジックを**ほぼ無改変で流用**できます。描画は floating origin が ECEF を正しく処理し、Havok は floating origin の region 機能によってステージ ECEF 近傍を float32 安全に解いてくれます。

:::note info
**`freezeWorldMatrix` を使っている理由**

`position` / `rotationQuaternion` / `scaling` を設定して Babylon に行列を組ませる、という普通のやり方をしていません。

X = East, Y = Up, Z = North という軸割り当ては `East × Up = -North`、すなわち `X × Y = -Z` の左手順序です。この基底を列に並べた行列の **行列式は -1（鏡映）** になりえます。鏡映を含む行列は TRS へ分解（decompose）できないため、`freezeWorldMatrix(world)` でワールド変換を直接固定しています。

なお det = -1 はこの軸割り当て自体に起因するもので、シーンが右手系（`useRightHandedSystem = true`）であることとは独立した話です。

この鏡映により `stageRoot` 配下の可視メッシュ（大砲）は面の winding が反転しうるのですが、実 GPU での目視確認では見えに問題がなく（砲弾は球なので不変、地形タイルは `stageRoot` 配下ではない）、物理も砲弾とコライダーが同一の鏡映ワールドを共有して自己整合するため、描画側の面反転補正は不要と判断しました。
:::

## 重力を「地球の中心向き」にする

平面シーンなら重力は `(0, -9.81, 0)` で済みますが、ECEF 空間では **「下」は場所ごとに違う方向**です。ステージ原点における「下」= ENU の Up の逆向きベクトル（ECEF）を Havok に渡します。

```ts
// src/demos/artillery/physics.ts
export const initPhysics = async (
    scene: Scene,
    gravity: Vector3 = new Vector3(0, DEMO_GRAVITY_Y, 0),
): Promise<HavokPlugin> => {
    const havokInstance = await HavokPhysics();
    const plugin = new HavokPlugin(true, havokInstance);
    plugin.setVelocityLimits(MAX_LINEAR_VELOCITY, MAX_ANGULAR_VELOCITY);
    scene.enablePhysics(gravity, plugin);
    return plugin;
};
```

呼び出し側は `stage.gravity`（= `frame.up.scale(DEMO_GRAVITY_Y)`）を渡すだけです。

```ts
// src/demos/artillery/index.ts
const stage: StageFrame = createStageFrame(scene, {
    lat: STAGE_CENTER.lat, lon: STAGE_CENTER.lon, alt: 0,
});
await initPhysics(scene, stage.gravity);
```

ステージが数 km 四方なので、その範囲内で重力方向を一定とみなす近似は十分に成立します（3km 離れても鉛直方向のズレは 0.03° 程度）。

## 「ワールド」と「ローカル」の使い分け

ここが実装で一番混乱しやすいポイントでした。Havok の API は**ワールド座標**で値を受け取る一方、メッシュの `position` は `stageRoot` の**ローカル座標**です。発射処理では両者を明示的に使い分けています。

```ts
// src/demos/artillery/index.ts
const fire = (): void => {
    const speed = powderToSpeed(Number(powderSlider.value));
    const cannon = gameState.turn === "red" ? redCannon : blueCannon;

    // 砲身の実際のワールド方向を取得（砲身ローカル +Y 軸が砲口方向）
    cannon.pivot.computeWorldMatrix(true);
    const worldDir = Vector3.TransformNormal(
        new Vector3(0, 1, 0),
        cannon.pivot.getWorldMatrix(),
    ).normalize();

    // ステージローカルの砲身方向。発射位置の算出に使う。
    const localDir = stage.worldDirToLocal(worldDir, new Vector3()).normalize();

    const pivotPos = cannon.pivot.position;
    // 発射位置（ステージローカル）: 砲口先端
    const launchPos = pivotPos.add(localDir.scale(BARREL_LENGTH));
    // 発射速度ベクトル（ワールド）: Havok の線形速度はワールド座標で与える。
    const velocity = worldDir.scale(speed);

    pool.acquire(launchPos, velocity);
};
```

| 値 | 座標系 | 理由 |
|---|---|---|
| `launchPos`（発射位置） | ステージローカル（ENU） | `mesh.position` に代入するため |
| `velocity`（初速） | ワールド（ECEF） | `body.setLinearVelocity()` がワールド指定のため |

プール側で `mesh.position` に入れてから `PhysicsAggregate` を生成し、ワールド速度を与えます。

```ts
// src/demos/artillery/projectilePool.ts
projectile.mesh.position.copyFrom(position);        // ← ローカル
projectile.mesh.rotationQuaternion = Quaternion.Identity();

const aggregate = new PhysicsAggregate(
    projectile.mesh,
    PhysicsShapeType.SPHERE,
    { mass: PROJECTILE_MASS, restitution: PROJECTILE_RESTITUTION, friction: PROJECTILE_FRICTION },
    scene,
);
aggregate.body.setLinearVelocity(velocity);          // ← ワールド
aggregate.body.setLinearDamping(0);                  // 空気抵抗で射程が縮まないように
aggregate.body.setAngularDamping(0);
```

---

# 地形コリジョン: ストリーミングされる地形にどう当てるか

## タイルに直接ボディを付けてはいけない

このプロジェクトの地形は**タイルストリーミング**で動的にロード・破棄・頂点更新されます。個々のタイルメッシュに `PhysicsAggregate` を付けると、タイルが差し替わるたびに物理ボディが陳腐化して非常に脆弱です。

そこで、**プレイエリアの可視地形を 1 枚のグリッドメッシュにサンプリングし、それにだけ静的ボディを付ける**という設計にしました。

```ts
// src/demos/artillery/terrainCollider.ts
export const DEFAULT_COLLIDER_OPTIONS: TerrainColliderOptions = {
    areaSize: 6000,      // 一辺 6km（半幅 3000m）
    subdivisions: 200,   // セル ≈ 30m
    restitution: 0.5,
    friction: 0.6,
};
```

`areaSize` は砲弾の最大射程から逆算しています。45° / 初速 600 m/s / 重力 180 m/s² で水平射程は `v²·sin(2θ)/g ≈ 2000m`。砲台がステージ原点から ±750m に置かれるので、最遠方位では着弾点が原点から約 2750m。余裕を見て半幅 3000m としています。

生成したコライダーメッシュは `stageRoot` に取り込みます。

```ts
const collider: TerrainCollider = createTerrainCollider(
    scene,
    undefined,
    (mesh) => stage.attach(mesh),   // ← stageRoot へ parent
);
```

コライダーは不可視ですが、ジオメトリは物理形状の元として保持します。

```ts
mesh.isVisible = false;
mesh.isPickable = false;
```

## 標高ダイレクト参照サンプラ

当初は頂点ごとにレイキャスト（`scene.pickWithRay`）して地表 Y を取っていましたが、`(200+1)² = 40,401` 頂点 × 全メッシュ走査は非常に重い処理でした。

最終的には、**レイキャストを使わず標高値を直接引く**方式に切り替えています。ここでも ENU ↔ ECEF の往復が効いてきます。

```ts
// src/demos/artillery/terrainSampler.ts
export const createDirectTerrainSampler = (
    stage: StageTransform,
    elevAt: (latDeg: number, lonDeg: number) => number | null,
): ((x: number, z: number) => number | null) => {
    const scratch = new Vector3();
    const scratchGeo: Geodetic = { latDeg: 0, lonDeg: 0, altMeters: 0 };
    return (x: number, z: number): number | null => {
        scratch.copyFromFloats(x, 0, z);
        stage.localToWorld(scratch, scratch);          // ① ENU → ECEF
        const geo = ecefToGeodeticToRef(scratch, scratchGeo);  // ② ECEF → 測地
        const elev = elevAt(geo.latDeg, geo.lonDeg);   // ③ 標高ルックアップ
        if (elev === null) return null;
        geodeticToEcefToRef(geo.latDeg, geo.lonDeg, elev, scratch); // ④ 測地 → ECEF
        stage.worldToLocal(scratch, scratch);          // ⑤ ECEF → ENU
        return scratch.y;
    };
};
```

この方式の副次的な利点として、**地球の曲率による落差が ④〜⑤ のローカル Y 変換に自然に織り込まれます**。平面近似の誤差を持ちません。ステージ端（3km 先）では曲率による落差が約 0.7m あるので、無視できない量です。

なお、この関数は約 4 万回呼ばれるため、`Vector3` と測地座標バッファは使い回してアロケーションを避けています。

## メインスレッドを止めない

4 万頂点のサンプリングを一気に回すとメインスレッドが長時間ブロックされ、ブラウザの「戻る」操作すら効かなくなります。フレーム時間予算ごとに `setTimeout(0)` で制御を返すようにしました。

```ts
let chunkStart = performance.now();
for (let i = 0; i < positions.length; i += 3) {
    const y = sampleY(positions[i], positions[i + 2]);
    if (y !== null) {
        positions[i + 1] = y;
        lastValidY = y;
        hitCount++;
    } else {
        // 未ロード等で取得失敗 → 直近の有効値で穴埋め（穴あきメッシュを避ける）
        positions[i + 1] = lastValidY;
    }
    if (performance.now() - chunkStart >= frameBudgetMs) {
        if (shouldAbort?.()) return null;
        await new Promise<void>((resolve) => setTimeout(resolve, 0));
        if (shouldAbort?.()) return null;
        chunkStart = performance.now();
    }
}

mesh.updateVerticesData(VertexBuffer.PositionKind, positions);
mesh.createNormals(false);
mesh.refreshBoundingInfo();

// 旧ボディを破棄して作り直す（頂点が変わったため形状を更新）
aggregate?.dispose();
aggregate = new PhysicsAggregate(mesh, PhysicsShapeType.MESH, { mass: 0, restitution, friction }, scene);
```

`shouldAbort` はページ離脱検知用のコールバックで、中断要求があれば即座に打ち切ります。

---

# 影を地形に落とす — これも座標系の話だった

大砲と砲弾の影を地形に落とす実装にも、この ENU/ECEF 変換が絡んでいます。

## DirectionalLight の方向・位置を ENU → ECEF へ写像する

「真上から照らす平行光源」を作りたいわけですが、ECEF 空間の `(0, -1, 0)` は「真上」ではありません。ステージの ENU Up を ECEF ベクトルとして求め、光源の方向と位置をそこへ写像します。

```ts
// src/demos/artillery/shadows.ts
const stageRoot = stage?.root ?? null;
let lightDir: Vector3;
let lightPos: Vector3;
if (stageRoot && stage) {
    // ローカル方向 (0.12,-1,0.1) を ENU basis で ECEF へ写像（原点差分で方向化）。
    const originW = stage.localToWorld(Vector3.Zero(), new Vector3());
    const tipW = stage.localToWorld(new Vector3(0.12, -1, 0.1), new Vector3());
    lightDir = tipW.subtract(originW).normalize();
    lightPos = stage.localToWorld(new Vector3(0, LIGHT_HEIGHT, 0), new Vector3());
} else {
    lightDir = new Vector3(0.12, -1, 0.1);
    lightPos = new Vector3(0, LIGHT_HEIGHT, 0);
}
const light = new DirectionalLight("artillery-top-light", lightDir, scene);
light.position = lightPos;
```

**方向ベクトルを「2点をローカル→ワールド変換してから差分をとる」形で求めている**のがポイントです。`TransformNormal` を使ってもよいのですが、平行移動成分を含む行列で方向を扱うときの取り違えを避けるため、原点との差分という明示的な形にしています。

ちなみに `(0, -1, 0)` ちょうどの完全な鉛直にしていないのは Babylon 側の事情です。真下すぎると `ShadowGenerator` のビュー行列の up ベクトルが退化してシャドウマップが壊れ、影が一切描画されません。約 8° の傾きを与えて回避しています（真下すぎる影は物体の真下に隠れて見えない、という UX 上の理由もあります）。

## ortho frustum を戦場サイズに固定する

`DirectionalLight` はデフォルトで `autoUpdateExtends = true` になっており、毎フレーム影キャスターのバウンディングボックスから ortho 範囲を自動計算します。ECEF スケールのシーンでこれを走らせると、範囲計算が不安定なうえ毎フレームのコストも無視できません。

戦場サイズが分かっているので、固定してしまいます。

```ts
const SHADOW_MAP_SIZE = 2048;
const FRUSTUM_RADIUS = 3000;   // 砲台 ±750m、射程 最大約2500m に対して余裕を持たせる
const LIGHT_HEIGHT = 8000;     // 砲弾の打ち上げ高度（最大約2000m）より十分高く

light.autoUpdateExtends = false;
light.autoCalcShadowZBounds = false;
light.shadowMinZ = 1;
light.shadowMaxZ = LIGHT_HEIGHT + 2000;
light.orthoLeft = -FRUSTUM_RADIUS;
light.orthoRight = FRUSTUM_RADIUS;
light.orthoTop = FRUSTUM_RADIUS;
light.orthoBottom = -FRUSTUM_RADIUS;
```

2048px で ±3000m をカバーするので約 3m/texel。砲台（直径約 80m）でも十分に精細な影が得られます。

## 影の受け手はストリーミングで増減する

地形タイルは動的に生成・破棄されるため、`receiveShadows` の設定も追随させる必要があります。毎フレーム全走査は無駄なので、差分スキャンにしています。

```ts
const receivers = new WeakSet<AbstractMesh>();
let lastScannedMeshCount = 0;

const registerTerrainReceivers = (): void => {
    const meshes = scene.meshes;
    // メッシュが dispose されて配列が縮むとインデックスがずれるため、
    // 縮小を検知したらスキャン位置をリセットして全走査し直す。
    if (meshes.length < lastScannedMeshCount) lastScannedMeshCount = 0;
    if (meshes.length <= lastScannedMeshCount) return;
    for (let i = lastScannedMeshCount; i < meshes.length; i++) {
        const mesh = meshes[i];
        if (!isTerrainReceiver(mesh.name)) continue;
        if (receivers.has(mesh)) continue;
        mesh.receiveShadows = true;
        receivers.add(mesh);
    }
    lastScannedMeshCount = meshes.length;
};
```

`WeakSet` で登録済みを弾きつつ、配列が縮んだら（＝タイルが dispose された）スキャン位置をリセットして取りこぼしを防ぎます。

---

# ハマったところ

## 速度上限に頭打ちされて飛ばない

Havok プラグインのデフォルト線形速度上限は約 200 です。砲弾の初速を最大 600 に設定しても、静かにクランプされて射程が出ませんでした。

```ts
// src/demos/artillery/physics.ts
export const MAX_LINEAR_VELOCITY = 5000;
export const MAX_ANGULAR_VELOCITY = 1000;

plugin.setVelocityLimits(MAX_LINEAR_VELOCITY, MAX_ANGULAR_VELOCITY);
```

エラーも警告も出ないので、「なぜか思ったより飛ばない」という形で現れます。デバッグ時は `setLinearVelocity` の直後に `getLinearVelocity()` を読んで、指定値と実際の値を突き合わせるログを仕込みました。

```ts
const applied = aggregate.body.getLinearVelocity();
console.debug(
    `[artillery] SET vel=(...) |v|=${velocity.length().toFixed(1)} ` +
    `→ applied=(...) |v|=${applied.length().toFixed(1)}`,
);
```

## 起動直後に大砲が地球の中心に描画される

大砲はステージローカル原点 `(0,0,0)` で生成され、地形ロード完了後の `placeCannons` まで正しい位置に配置されません。平面シーンなら「原点にある」だけで済みますが、**ECEF では stageRoot のローカル原点は正しい場所（ステージ中心の海抜0m）に写像される**ので、これ自体は問題ないはずでした。

実際に起きたのは、`stageRoot` に parent される前の数フレームで大砲が ECEF 原点（＝地球の中心）付近に描画され、画面端で一瞬フラッシュするという現象です。生成直後に `setEnabled(false)` して、初回 `placeCannons` 後に有効化することで解消しました。

Geospatial に移行すると「原点＝画面中央」ではなくなるので、初期化順序に起因する一瞬の表示が目立つようになります。

## 重力は現実の 9.81 にしていない

これは精度と直接は関係ありませんが、大砲・砲弾を現実よりずっと大きいスケールで表示しているため、`g = -9.81` だと放物線が間延びして気持ちよくありません。

```ts
/**
 * 射程調整メモ:
 * - g=-150 (初期値): Power=100% の射程約3398 → 大砲間距離(約1288)を大きくオーバーシュート
 * - g=-250: 射程約2040 → 若干強すぎて地形によっては届かない
 * - g=-180 (現値): flat R = v²/|g| = 600²/180 ≈ 2000。
 *   Power=100% でギリギリ届く難易度に調整。
 */
export const DEMO_GRAVITY_Y = -180;
```

「実測して数値を決め、根拠をコメントに残す」というのは、この手のパラメータでは特に重要だと感じています。

---

# まとめ

Babylon.js 9 の Geospatial（float64 / ECEF / floating origin）の上で、float32 ベースの Havok 物理を成立させるための要点です。

1. **物理をワールド（ECEF）で解こうとしない。** ECEF の絶対座標は float32 で 0.5m 以上に量子化されるので、剛体位置をそのまま渡す設計は破綻する。
2. **ステージ原点に ENU フレームを張り、`stageRoot`（`TransformNode`）に全メッシュをぶら下げる。** 子はローカル座標（±3000m）で扱えるので float32 で十分な精度が出る。描画は floating origin が ECEF を処理する。
3. **重力は ENU の Up の逆向き（ECEF ベクトル）を渡す。** `(0,-1,0)` は ECEF では「下」ではない。
4. **API ごとの座標系を意識する。** `mesh.position` はローカル、`body.setLinearVelocity()` はワールド。
5. **ストリーミングされる地形には直接ボディを付けない。** プレイエリアを 1 枚のグリッドにサンプリングして静的ボディを付け、ストリーミングから分離する。
6. **光源も同じ変換が要る。** `DirectionalLight` の方向・位置を ENU → ECEF へ写像し、ortho frustum は戦場サイズに固定する。

「地球規模の座標系」と「局所的な物理シミュレーション」は、素直に接続できません。**その境界に薄い抽象（`StageFrame`）を1枚挟む**ことで、ゲームロジック側はほぼ何も知らずに済むようにできた、というのが今回の設計の勘所でした。

そして最後に、記事中で可視化してみたように、**この座標基盤は普段まったく目に見えません**。見えないものを扱うときこそ、デバッグ用に描いて確かめるのが結局いちばん早い、というのも今回得た教訓でした。

## 余談: 影が出なかった本当の理由

実は影の実装で最初に苦戦したのは、上に書いた座標変換ではなく、 **「影を持たない主平行光がシーンに既に存在していて、影の領域を明るく塗りつぶしていた」** という、Geospatial とは無関係な問題でした。加えて WebGPU では PCF フィルタ（`usePercentageCloserFiltering`）を使うとシーン全体が白画面になるという別の罠もあり、Poisson sampling に落ち着いています。

---

# リポジトリ

- https://github.com/8ga3/jpmap_terrain
- 地理院タイルの標高データから 3D 地形を生成する Babylon.js / TypeScript のライブラリ + デモ集です

# 参考

- [Babylon.js Documentation](https://doc.babylonjs.com/)
- [地理院タイル](https://maps.gsi.go.jp/development/ichiran.html)
