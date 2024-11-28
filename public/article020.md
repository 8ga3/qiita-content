---
title: Babylon.jsでイージングを組み合わせたアニメーション
tags:
  - JavaScript
  - Babylon.js
private: true
updated_at: '2024-11-28T11:49:30+09:00'
id: 630765c466c7bff8d2ff
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

2024年11月に行われていた[技術書典17](https://techbookfest.org/event/tbf17)で、[Babylon.js 勉強会](https://techbookfest.org/organization/dqewxtYMsQVP3EiYjeQyCH)さんの **Babylon.js レシピ集 Vol.1~5** を購入しました。

一通り目を通し、次は自分で色々試してみたので、そこで知ったことをTipsとして書きました。

# イージングについて

前記レシピ集の執筆陣になっている方の[解説](https://qiita.com/youtoy/items/5c8c3cebeed1e2285616)を参照してください。
Unityでは[アニメーションカーブ](https://docs.unity3d.com/ja/2022.3/Manual/animeditor-AnimationCurves.html)をイメージしてもらえればわかりやすいと思います。

簡単に言うと、移動のような動きや色の変化など緩急をつけてあげる機能です。

https://qiita.com/youtoy/items/5c8c3cebeed1e2285616


# イージングしながらノードを往復させる

![UnityAnimePanel.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/3774e525-7b68-368b-22dd-f3e64ef6322f.png)

Unityのアニメーションのパネルです。こんな感じの動きをさせようと思います。


# 公式サンプル

公式の[サンプル](https://playground.babylonjs.com/#8ZNVGR)を一部抜粋します。

`keysBezierTorus`にキーフレームの座標を設定しています。イージングは`setEasingFunction()`に設定しています。この方法ですと、120フレーム目の場所に移動したあと、0フレームに戻ってリピートするため見た目がワープしてしまいます。

```javascript
    // Torus
    var bezierTorus = BABYLON.Mesh.CreateTorus("torus", 8, 2, 32, scene, false);
    bezierTorus.position.x = 25;
    bezierTorus.position.z = 0;

    // Create the animation
    var animationBezierTorus = new BABYLON.Animation("animationBezierTorus", "position", 30, BABYLON.Animation.ANIMATIONTYPE_VECTOR3, BABYLON.Animation.ANIMATIONLOOPMODE_CYCLE);
    var keysBezierTorus = [];
    keysBezierTorus.push({ frame: 0, value: bezierTorus.position });
    keysBezierTorus.push({ frame: 120, value: bezierTorus.position.add(new BABYLON.Vector3(-80, 0, 0)) });
    animationBezierTorus.setKeys(keysBezierTorus);
    var bezierEase = new BABYLON.BezierCurveEase(0.32, -0.73, 0.69, 1.59);
    animationBezierTorus.setEasingFunction(bezierEase);
    bezierTorus.animations.push(animationBezierTorus);
    scene.beginAnimation(bezierTorus, 0, 120, true);
```

# サンプルコードを改変して往復させてみる

往復するには、キーフレームの定義のほうにイージングを指定することで可能です。
先に`easingFunction`を作っておきます。ここでは`BABYLON.SineEase()`にしてみました。
`setEasingMode`で`BABYLON.EasingFunction.EASINGMODE_EASEINOUT`にしました。ゆっくり動き出し、中間で一番速くなり、ゆっくり止まるようになります。

つぎに`keysTorus`にキーフレームを定義します。このとき、`easingFunction`に`easingFunction`を設定することができます。ここでは、0フレームにイージングを設定、移動先の120フレーム目で一旦止まるのでまたイージングを設定しています。240フレームで最初の場所に戻ります。

[Playgroundの方](https://playground.babylonjs.com/#2CTZW8)に上げたので、よければイージングの種類を変えたりキーフレームを増やしたりして試してみてください。

![swing.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/ff2ea4aa-6dd9-91a3-0802-754464d673fe.gif)

```javascript
const canvas = document.getElementById("renderCanvas");
const engine = new BABYLON.Engine(canvas, true);

const createScene = () => {
    var scene = new BABYLON.Scene(engine);

    var camera = new BABYLON.ArcRotateCamera("Camera",
        0, 0.8, 100, BABYLON.Vector3.Zero(), scene);

    camera.attachControl(canvas, true);

    const light = new BABYLON.HemisphericLight("light",
        new BABYLON.Vector3(1, 1, 0), scene);

    // Torus
    var torus = BABYLON.MeshBuilder.CreateTorus("torus",
        {thickness: 2, tessellation: 32, diameter: 8}, scene);
    torus.position.x = 25;
    torus.position.z = 30;

    const animationTorus = new BABYLON.Animation("torusEasingAnimation",
        "position", 30, BABYLON.Animation.ANIMATIONTYPE_VECTOR3,
        BABYLON.Animation.ANIMATIONLOOPMODE_CYCLE);

    const easingFunction = new BABYLON.SineEase();
    easingFunction.setEasingMode(BABYLON.EasingFunction.EASINGMODE_EASEINOUT);

    // Animation keys
    const nextPos = torus.position.add(new BABYLON.Vector3(-80, 0, -80));

    const keysTorus = [];
    keysTorus.push({ frame: 0,   value: torus.position, easingFunction: easingFunction });
    keysTorus.push({ frame: 120, value: nextPos,        easingFunction: easingFunction });
    keysTorus.push({ frame: 240, value: torus.position });
    animationTorus.setKeys(keysTorus);

    scene.beginDirectAnimation(torus, [animationTorus], 0, 240, true);

    return scene;
}

const scene = createScene();
engine.runRenderLoop(() => { scene.render();});
window.addEventListener("resize", () => { engine.resize(); });
```

https://playground.babylonjs.com/#2CTZW8

# さいごに

メダル落としを作ってみました。プッシャーが前後に動くところにイージングを使ってます。

ちなみに、AndroidのChromeでARモードにすることができます。 先に **chrome://flags** でWebXRの設定を有効にする必要があるかもしれません。

![PushMedal.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e0724ffb-b6ce-682c-4f85-2f0436fc24a9.png)

https://playground.babylonjs.com/#579QOL#5
