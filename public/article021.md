---
title: Babylon.jsを使ってPLYファイルの点群を簡単に表示する方法
tags:
  - 'Babylon.js'
  - 'JavaScript'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

![kushikatsu.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/545d1829-60ef-4b59-16aa-3574d2cd4ebe.png)

Babylon.jsには点群（ポイント・クラウド）を扱う機能として、[ポイント・クラウド・システム](https://doc.babylonjs.com/features/featuresDeepDive/particles/point_cloud_system/)があります。十分な機能を持っていて便利だと思います。

ポイント・クラウド・システムは、メッシュ（UnityのGameObject相当）を生成したりすることはできます。しかし、点群ファイルの読み書きに対応したファンクションが見当たりませんでした。

そう、Version 7.0より前の話です。

# 発見

PLYとかPCDなどの点群が扱えるフォーマットのファイルを読むためのファンクションを自前で作るしかないかなと思いつつ、ちょっとだるいなーと思っていました。

ふとBabylon.js 7.0の新機能の紹介ムービーを見直していて、 **3D Gaussian Splatting** が目に入ってきました。3DGSを保存しているファイルフォーマットはなんだろうか、ドキュメントの方をみてみました。

[Gaussian Splatting](https://doc.babylonjs.com/features/featuresDeepDive/mesh/gaussianSplatting/)のページを見ると、 **.PLY** と **.splat** が対応とありました。

PLYフォーマットの3DGSなら、iOSアプリの[Scaniverse](https://apps.apple.com/jp/app/scaniverse-3d-scanner/id1541433223)でエクスポートできます。Nianticが開発しているし無料だしで素晴らしいアプリです。

3DGSのPLYファイルが読み込めるなら、もしかしたら普通の点群も読めるのではと考えました。そこでGitHubのソースコードを確認しに行ってみました。

なんと予想はあたって、 **ちゃんと点群も読み込めるようにコーディングされていました！** 3DGS・点群と、メッシュ（ポリゴン）の3種類に対応です。[splatFileLoader.ts](https://github.com/BabylonJS/Babylon.js/blob/3725abaed67d26b5a8de053f3aa336deae4373b1/packages/dev/loaders/src/SPLAT/splatFileLoader.ts#L182)が該当箇所です。

ソースコードを読むと、点群は点のXYZ座標とRGBカラーを読み込んでいて、その他の要素があった場合は無視していました。法線ベクトルやインテンシティといった要素がファイル内にあって、もし必要であればsplatFileLoader.tsを拡張するとか、別途ローダーをコーディングするとかが必要になりますのでご注意ください。

# PLYファイル読み込み方法

babylon.jsを実行できるように、HTMLファイルに書いておきます。このとき、ファイルのローディング部分は **babylonjs.loaders.min.js** のほうになるので、これも追加しておきます。

なおCDNに公開されたjsファイルは、本番環境では使わないようにしましょう。[こちら](https://doc.babylonjs.com/setup/frameworkPackages/CDN/)に注意書きがありますので、合わせて参照してください。

```html:index.html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <title>Babylon.js Point Cloud Viewer</title>

    <!-- Babylon.js -->
    <script src="https://cdn.babylonjs.com/babylon.js"></script>
    <script src="https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"></script>

    <link rel="stylesheet" type="text/css" href="style.css" />
  </head>
  <body>
    <canvas id="renderCanvas"></canvas>
    <script src="index.js"></script>
  </body>
</html>
```

CSSファイルはこんな感じで、特に説明はありません。

```css:style.css
html, body {
    overflow: hidden;
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
}
#renderCanvas {
width: 100%;
    height: 100%;
}
```

glTF・OBJ・STLなどをロードするときと同じように `BABYLON.SceneLoader.ImportMeshAsync()` を使います。PLYファイルはサーバーに置くなりして、任意のURLを指定します。非同期読み込みなので、`const createScene = async () => {}`のように`async`をつけておきましょう。

読み込んだあと、`then()`に渡したアロー関数が実行されます。`result.meshes`に点群が入っていますが、配列なので0番目の要素をとりだします。

`mesh.material.pointSize`で点の大きさを設定しています。これはお好みでOKです。

**Scaniverse** でエクスポートしたPLYファイルはZ軸が上方向でした。X軸を中心に90度回転し、Y軸が上になるようにしています。

原点が点群の端の方にあったりするので、点群の中心がワールド座標の中心になるように移動させています。

```javascript:index.js
const canvas = document.getElementById("renderCanvas");
const engine = new BABYLON.Engine(canvas, true);

// シーンの作成
const createScene = async () => {
    const scene = new BABYLON.Scene(engine);
    // XYZ軸を表示したいときはコメントを外す
    // new BABYLON.AxesViewer(scene, 0.1);

    const camera = new BABYLON.ArcRotateCamera("Camera", 0, Math.PI / 4, 0.5, BABYLON.Vector3.Zero(), scene);
    camera.wheelPrecision = 300;
    camera.inertia = 0.9;
    camera.minZ = 0.01;
    camera.attachControl(canvas, true);

    BABYLON.SceneLoader.ImportMeshAsync(null, "", "res/kushikatsu.ply", scene).then((result) => {
        const mesh = result.meshes[0];
        mesh.material.pointSize = 1.5;
        // 向きと位置を修正
        const center = BABYLON.Mesh.Center(result.meshes);
        mesh.rotateAround(center, new BABYLON.Vector3(1, 0, 0), -Math.PI / 2);
        mesh.position.subtractInPlace(center);
    });

    scene.onBeforeRenderObservable.add(() => {
        camera.beta = Math.min(camera.beta, Math.PI / 2);
        camera.radius = Math.max(camera.radius, 0.2);
        camera.radius = Math.min(camera.radius, 2.);
    });

    return scene;
};

// シーンの作成とレンダリング
createScene().then((scene) => {
    // デバッグ用のパネルを表示したいときはコメントを外す
    // scene.debugLayer.show();
    engine.runRenderLoop(function () {
        if (scene) {
            scene.render();
        }
    });
});

// ウィンドウリサイズ時の処理
window.addEventListener("resize", () => { engine.resize(); });
```

# 実行

テストで使ったPLYファイルは串カツです。

ポイント数ですが、ちょっと試した感じですと100万ポイントは表示できました。1000万ポイントはChromeのコンソールにメモリエラーがでて動きませんでした。WebブラウザのWebGLがエラーを出しているのか、javascriptエンジンがエラーを出しているのか、PCのメモリ搭載量の問題なのかは不明です。

![kushikatsu.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/0af954ef-af48-7b4a-2c3b-4638324cbf28.gif)

当方のGitHubの[リポジトリ](https://github.com/8ga3/babylon_KushikatsuPC)も用意しましたので活用していただければ幸いです。表示方法はBabylon.jsのチュートリアルなどご参照ください。

https://github.com/8ga3/babylon_KushikatsuPC
