---
title: Meta Quest Browserのコントローラは「押していないのにタッチ」——WebアプリからみたQuestのポインタ挙動
tags:
  - jpmap_terrain
  - WebXR
  - MetaQuest
  - PointerEvents
  - TypeScript
private: false
updated_at: ''
id: 79a1e61b22c7eb824687
organization_url_name: access
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

jpmap_terrainについては、こちらの記事で紹介しています。

https://qiita.com/8ga3/items/33c1b135fcebd3ec8107

地理院タイルの標高データから地形を生成するWebアプリ [jpmap_terrain](https://github.com/8ga3/jpmap_terrain) に、地形を手元サイズの正方形として表示する「箱庭ジオラマ」というデモがあります。Babylon.js + TypeScript製で、WebXR (`immersive-ar`) にも対応しており、Meta Quest 3 で実機検証しながら作りました。

このデモの入力まわりを実装していて一番手間取ったのが、**Meta Quest Browser のコントローラーがWebページに届けてくるポインタイベントが、マウスともスマホのタッチとも違う**という点でした。「PCで動いたからQuestでも大丈夫だろう」と思っていると、だいたい足をすくわれます。

この記事では、Quest Browser のコントローラのポインタ挙動を整理し、Web側でどう受け止めればいいかをまとめます。

![metaquest.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/d8ced9c8-f4a2-4746-9394-78404199fc93.jpeg)

## まず全体像：Questには2つの入力レイヤーがある

Quest 向けのWebアプリを書くとき、コントローラーの入力経路は**2つ**あり、これを混同すると話が噛み合わなくなります。

| | 経路 | 入力の取り方 | いつ使う |
|---|---|---|---|
| ① | **DOMポインタイベント** | `pointerdown` / `pointermove` / `pointerup` | 通常のWebページ表示中（2Dブラウザウィンドウをレイで操作） |
| ② | **WebXR input sources** | `XRSession` 経由。スティック軸やボタンの値を毎フレーム読む | `immersive-vr` / `immersive-ar` セッション中 |

**①は「レイでブラウザ画面をポイントしてトリガーを引く」操作**です。ページからは普通のポインタイベントに見えます。ボタンクリックやドラッグはここで処理します。

**②は没入セッションに入ってから**の話で、こちらは**スティックやトリガーの生の値**が取れます。DOMのポインタイベントとは完全に別物です。

この記事で主に扱うのは **①のクセ**です。②については最後に少し触れます。

## Quest Browser のポインタ挙動、5つのクセ

### クセ1: コントローラーのレイは `pointerType === "touch"`

まずここです。直感的には「レイポインタ＝マウスカーソルみたいなもの」なので `"mouse"` を期待しますが、Quest Browser では **`pointerType` が `"touch"`** として届きます。

```ts
element.addEventListener("pointerdown", (e) => {
    console.log(e.pointerType); // Quest のコントローラー: "touch"
});
```

`pointerType === "mouse"` で分岐して「マウス用の処理」を書いていると、Quest では**まるごとスキップされます**。逆に「タッチならモバイル向けUIを出す」という分岐も、Quest では意図せず発動します。

### クセ2: トリガーを引いていなくても `pointermove` が飛んでくる

ここが一番の特徴です。**コントローラーのレイを画面に向けているだけで `pointermove` が継続的に発生します。** トリガーは引いていません。

つまり **「ホバーがあるタッチ」** という、通常のタッチスクリーンには存在しない状態が生まれます。スマホのタッチは「指が触れている＝押している」なので、`pointermove` が来た時点で接触中と見なせますが、**Questではその前提が成立しません**。

見分けるには `buttons` を見ます。

```ts
element.addEventListener("pointermove", (e) => {
    if (e.buttons === 0) {
        // レイを向けているだけ（ホバー）。まだ押されていない。
        return;
    }
    // 実際にトリガーを引いた状態でのドラッグ
});
```

`PointerEvent.buttons` は押下中のボタンのビットマスクで、**`0` なら何も押されていない**ことが確実にわかります。「タッチだから押されているはず」ではなく、**`buttons` を見て判定する**のが正解です。

### クセ3: ホバーのポインタに `pointerup` は来ない

クセ2と対になる話です。ホバーで発生した `pointermove` の `pointerId` に対して、**`pointerup` は永久に来ません**。当然で、押していないのだから離すこともないからです。

これが厄介なのは、`pointerdown` → `pointerup` の対応を前提にした状態管理が壊れることです。

```ts
// これは Quest では破綻しうる
const activePointers = new Set<number>();
el.addEventListener("pointermove", (e) => activePointers.add(e.pointerId)); // ホバーで入る
el.addEventListener("pointerup", (e) => activePointers.delete(e.pointerId)); // 永久に呼ばれない
```

`pointermove` を見て「押下中」と推定する実装（`pointerdown` を取りこぼしたときの救済としてよくあるパターン）を書くと、ホバー由来のIDが**残り続けて消えなくなります**。

対策としては、

- 押下状態の追跡は原則 `pointerdown` を起点にする
- `pointermove` / `pointerover` / `pointerout` で **`buttons === 0` を観測したら、そのポインタは「押されていない」と確定して追跡から外す**

の2点です。後者は「離した通知が来ない」問題に対する実質的な代替になります。

### クセ4: `pointerId` は操作のたびに変わる

同じコントローラーで2回タップしても、**`pointerId` は別の値になります**。マウスのように「ポインタは1つで固定ID」ではありません。

なので、

- **`pointerId` を長期的な識別子として保存しない**（「このIDは左コントローラー」といった覚え方はできない）
- ドラッグの追跡には使えるが、**そのドラッグの間だけ**有効なものとして扱う

という前提で書きます。ジョイスティックUIの実装ではこうしました。

```ts
let activePointerId: number | null = null;

const onPointerDown = (event: PointerEvent): void => {
    if (activePointerId !== null) return;        // 既にドラッグ中なら無視
    activePointerId = event.pointerId;
    outer.setPointerCapture?.(event.pointerId);  // 要素外へ出てもドラッグ継続
    updateFromClientPoint(event.clientX, event.clientY);
};
const onPointerMove = (event: PointerEvent): void => {
    if (event.pointerId !== activePointerId) return;  // 他のポインタは無視
    updateFromClientPoint(event.clientX, event.clientY);
};
const onPointerUp = (event: PointerEvent): void => {
    if (event.pointerId !== activePointerId) return;
    activePointerId = null;
    resetKnob();
};

outer.addEventListener("pointerdown", onPointerDown);
outer.addEventListener("pointermove", onPointerMove);
outer.addEventListener("pointerup", onPointerUp);
outer.addEventListener("pointercancel", onPointerUp); // cancelも必ず拾う
```

ポイントは3つです。

- **`setPointerCapture`**: ノブが要素の外に出てもイベントを受け取り続けられます。これが無いと、少し大きくドラッグしただけでスティックが飛びます。
- **`pointerId` の一致チェック**: 左右のコントローラーや複数指が絡んでも、他のポインタの `pointerup` で自分の押下状態が誤解除されません。
- **`pointercancel` も `pointerup` と同じ扱い**: cancel は普通に飛んできます。拾わないと「押しっぱなし」で固まります。

### クセ5: A/Xボタン＋コントローラー移動が「マウスのドラッグ」に化ける

Quest Browser では、コントローラーの **A/Xボタンを押しながらコントローラーを動かすと、マウスの「押下＋ドラッグ」として合成される**挙動があります。

3Dビューアとしては何気ない操作なのですが、これがページ上で起きると**テキスト選択（ドラッグ選択）が発動します**。UIボタンの文字列が青くハイライトされ、見た目が壊れます。

3DCGビューアで選択可能なテキストを提供する必要はないので、ページ全体で無効化しました。

```css
html, body {
  /* Meta Quest Browser等、A/Xボタン押下＋コントローラー移動をマウスの
     「押下＋ドラッグ」として合成する環境では、ページ上のテキストが
     意図せず選択されてしまう。 */
  user-select: none;
  -webkit-user-select: none;
}
```

ついでに、ブラウザ既定のジェスチャも切っておくと安定します。

```css
html, body {
  overflow: hidden;
  touch-action: none;        /* 2本指スクロール・ダブルタップズーム等を抑止 */
  overscroll-behavior: none;
}
```

`pointerType` が `"touch"` である以上、**ブラウザはタッチ向けのジェスチャ処理を適用してきます**。`touch-action: none` はQuestでも効きます。

## 没入セッション中（immersive-ar / immersive-vr）は話が変わる

WebXRの没入セッションに入ると、**画面はブラウザのページではなくXRの描画に切り替わり、DOMのポインタイベントは基本的に来なくなります**。ここからは②の世界で、スティックやトリガーの値を毎フレーム読むことになります。

Babylon.js の場合はこうです。

```ts
const thumbstick = motionController.getComponentOfType("thumbstick");
if (thumbstick) {
    thumbstick.onAxisValueChangedObservable.add(({ x, y }) => {
        sticks[handedness] = { x: clamp1(x), y: clamp1(y) };
    });
}
const trigger = motionController.getComponentOfType("trigger");
```

A/B/X/Y ボタンは、WebXR入力プロファイル（`@webxr-input-profiles/motion-controllers` の oculus-touch系プロファイル）のコンポーネントIDで取れます。

```ts
const PRIMARY_BUTTON_COMPONENT_ID = { left: "x-button", right: "a-button" };
const SECONDARY_BUTTON_COMPONENT_ID = { left: "y-button", right: "b-button" };
```

`getComponent(id)` は該当が無ければ `undefined` を返すので、**プロファイルによって命名が違う場合に備えてインデックス指定のフォールバックを用意しておく**と安全です。

### 没入中にDOMのUIを出したいなら `dom-overlay`

「没入中もHTMLのボタンを出したい」場合は WebXR の `dom-overlay` feature を使います。ここで一点、仕様上のハマりどころがあります。

> **`dom-overlay` は `requestSession` の時点で `XRSessionInit` に含まれていないと反映されない。**

つまり、**セッション開始より前**に feature を有効化しておく必要があります。後から登録しても、ライブラリ的にはエラーなく成功したように見えるのに、**実機では没入中にGUIが一切表示されません**。Babylon.js なら `enterXRAsync` より前に `enableFeature(WebXRFeatureName.DOM_OVERLAY, ...)` を呼びます。

加えて、`domOverlay.root` に渡す要素は **`requestSession` の時点で文書に接続済み（`Node.isConnected`）** である必要があります。未接続の要素を渡すと `requestSession` 自体が失敗し、AR突入直後に通常表示へ戻される、という挙動になります。

このため、実装は「HUD生成 + feature登録（セッション開始前）」と「毎フレームの入力反映（セッション開始後）」に分けています。

### スティック特有の話：斜めのズレ

これはQuest固有というより物理スティック全般の話ですが、実装上ハマるので触れておきます。

右スティックの X を回転、Y をズームに割り当てたところ、 **「下に倒してズームしているつもりが、わずかな左右のズレで回転も一緒に発動する」** という問題が出ました。物理スティックは真っ直ぐ倒すのが意外と難しいためです。

対策として、**支配的な軸だけを有効にする「十字キー化」のゲート**を挟んでいます。X と Y を素直に独立して扱わず、`|x| > |y|` なら X のみ、逆なら Y のみを通す、という処理です。オンスクリーンのボタンUIは元から排他なので、この処理は物理スティックの入力にだけ適用します。

## 実機検証について（別記事）

ここまでのクセは、どれも**実機で1回ログを出せば即座にわかる**類のものです。逆に言えば、実機で確認する手段を持たないまま推測でコードを書くと延々ハマります。

ただ、Quest でのWeb開発は手元で確認するまでのハードルが少し高く、

- **WebXR はセキュアコンテキスト必須**なので、`http://192.168.x.x:5173/` のようなLAN内の平文HTTPでは WebXR API が使えない（`localhost` は例外だが、Quest から見た `localhost` は Quest 自身）
- **Quest では console のログが見えにくい**ため、`pointerType` / `buttons` / `pointerId` をどう目視するかを先に決めておく必要がある

といった前準備が要ります。

アプローチは大きく2通りあり、このプロジェクトでは状況に応じて両方使いました。

**① 開発サーバをHTTPSで公開して、Questのブラウザから開く**
トンネルサービス等で一時的なHTTPS URLを払い出す方法です。ケーブル不要で手軽な反面、URLの共有や有効期限の扱いが少し面倒です。

**② USBケーブルでPCと繋いで `adb` を使う**
`adb reverse` でPCの開発サーバをQuest側の `localhost` に転送すると、**`localhost` はセキュアコンテキスト扱いになるため、HTTPSを用意しなくてもWebXRが動きます**。さらに `adb logcat` やリモートデバッグでPC側からログを直接拾えるので、「Questでは console が見えにくい」問題もそのまま解決します。今回のようにポインタイベントを1件ずつ観察したい場面では、こちらが圧倒的に速かったです。

このHTTPS公開手順については、こちらのページにまとめたのでご覧ください。

https://qiita.com/8ga3/items/7d4d2ae52acf441a5993

## まとめ：チェックリスト

Quest Browser 向けにポインタ入力を書くときの確認事項です。

- [ ] `pointerType === "mouse"` で分岐していないか（Questは **`"touch"`**）
- [ ] `pointermove` を「接触中」と見なしていないか（**`buttons === 0` のホバーがある**）
- [ ] `pointerdown` と `pointerup` が必ず対で来る前提になっていないか（**ホバーにupは来ない**）
- [ ] `pointerId` を長期的な識別子として保存していないか（**毎回変わる**）
- [ ] ドラッグに `setPointerCapture` を使っているか
- [ ] `pointercancel` を `pointerup` と同じく処理しているか
- [ ] `user-select: none` / `touch-action: none` を当てているか
- [ ] 没入セッション中の入力は WebXR input sources から取っているか
- [ ] `dom-overlay` をセッション開始**前**に有効化しているか
- [ ] 実機検証の手段を用意したか（HTTPSトンネル、または USB+`adb`）

## おわりに

「WebXR対応」というと描画やパフォーマンスの話に注目しがちですが、実際に体験の質を落とすのは**入力まわりのプラットフォーム固有の細かい差**だったりします。

Quest のコントローラは「マウスのようでマウスではなく、タッチのようでタッチでもない」という中間的な存在で、**ホバーできるタッチ**という点が最大の特徴です。この1点を頭に入れておくだけで、原因不明の入力バグの多くは説明がつくようになります。

Quest で入力が怪しいときは、まず `pointerType` と `buttons` と `pointerId` を実機で出してみてください。だいたいそこに答えがあります。

- リポジトリ: https://github.com/8ga3/jpmap_terrain
- 箱庭ジオラマデモの仕様: [spec/demos.md](https://github.com/8ga3/jpmap_terrain/blob/main/spec/demos.md)
- 関連記事: WebXRのジオラマが動かない？ セキュアコンテキストとHTTPSの話 ※公開後にリンクを貼ります
