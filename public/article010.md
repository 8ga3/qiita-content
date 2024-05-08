---
title: UnityのInput SystemでUSB接続のゲームパッドを使用できるようにする手順
tags:
  - Unity
  - macOS
  - Logicool
  - F310
private: false
updated_at: ''
id: null
organization_url_name: access
slide: false
ignorePublish: false
---
# 概要

[Logicool F310](https://gaming.logicool.co.jp/ja-jp/products/gamepads/f310-gamepad.940-000137.html)というUSB接続のゲームパッドを持っています。

Unityプログラミングをしていて、ふとゲームパッドを使ってみようかと思ったところ使えるようになるまで色々あったので、ここに記録しておきたいと思います。

:::note warn
いまはLogicool F310rという名前でマイナーモデルチェンジしているようです。
HID のProduct IDが記事とは違う可能性があります。
:::

# 開発環境

* macOS Sonoma 14.4.1 (Intel & Apple silicon)
* Unity 2022.3.27f1

おそらくWindowsでも同じような感じになります。

# ゲームパッド接続

F310をMacにUSB type-cに変換して接続します。このとき背面のスイッチは **D** のDirectInputインターフェイスモードにします。XInputインターフェイスモードは使用できません。

初めてMacに接続したとき認識せず、数回USBコネクタを抜き差しが必要でした。（IntelとApple siliconの2台とも同じ状況でした。なぜ？？）接続できているかは **システム情報** のハードウェア、USBの項目で確認できます。

![article010_01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/b971199b-7fd5-3950-2281-bad0de066dc9.png)

認識されるとこんな感じになります。一度認識できれば、次からはすぐ使えるようです。

**システム設定** でも確認可能です。私は **Lanchpadを開くにはホームボタンを押します** をOFFにしています。F310でBackボタンを押すとLanchpadへ画面遷移してしまって邪魔なのです。

![article010_02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/69e48650-a8f4-df51-c270-3b45fdfd9cb7.png)

<details><summary>さらにターミナルからdefaultsコマンドで確認できます。</summary>

``` console
% defaults read com.apple.GameController
{
    bluetoothPrefsMenuLongPressAction = 0;
    controllers =     {
        data =         (
                        {
                baseProfile =                 {
                    customizable = 1;
                    "doublePressShareGesture_mac" = 1;
                    elementMappings =                     {
                    };
                    hapticFeedbackOverride = 0;
                    hapticStrength = 1;
                    isBaseProfile = 1;
                    lightbarColor = 0;
                    lightbarCustomColorEnabled = 0;
                    lightbarOverride = 0;
                    "longPressShareGesture_mac" = 2;
                    modifiedDate = "2024-05-08 18:25:35 +0900";
                    name = "Base Profile";
                    sfSymbolsName = "gamecontroller.fill";
                    uuid = "317D6688-3903-4FD4-90A1-2CA494A61BF8";
                };
                buttons =                 {
                    "Button A" =                     {
                        kind = 1;
                        name = "Button A";
                        nameLocalizationKey = "BUTTON_A";
                        remappingKey = 4;
                        sfSymbolsName = "a.circle";
                    };
                    "Button B" =                     {
                        kind = 1;
                        name = "Button B";
                        nameLocalizationKey = "BUTTON_B";
                        remappingKey = 5;
                        sfSymbolsName = "b.circle";
                    };
                    "Button X" =                     {
                        kind = 1;
                        name = "Button X";
                        nameLocalizationKey = "BUTTON_X";
                        remappingKey = 6;
                        sfSymbolsName = "x.circle";
                    };
                    "Button Y" =                     {
                        kind = 1;
                        name = "Button Y";
                        nameLocalizationKey = "BUTTON_Y";
                        remappingKey = 7;
                        sfSymbolsName = "y.circle";
                    };
                    "Left Shoulder" =                     {
                        kind = 1;
                        name = "Left Shoulder";
                        nameLocalizationKey = "LEFT_SHOULDER";
                        remappingKey = 8;
                        sfSymbolsName = "l1.rectangle.roundedbottom";
                    };
                    "Left Thumbstick Button" =                     {
                        kind = 1;
                        name = "Left Thumbstick Button";
                        nameLocalizationKey = "BUTTON_LEFT_THUMBSTICK";
                        remappingKey = 20;
                        sfSymbolsName = "l.joystick.press.down";
                    };
                    "Left Trigger" =                     {
                        kind = 1;
                        name = "Left Trigger";
                        nameLocalizationKey = "LEFT_TRIGGER";
                        remappingKey = 18;
                        sfSymbolsName = "l2.rectangle.roundedtop";
                    };
                    "Right Shoulder" =                     {
                        kind = 1;
                        name = "Right Shoulder";
                        nameLocalizationKey = "RIGHT_SHOULDER";
                        remappingKey = 9;
                        sfSymbolsName = "r1.rectangle.roundedbottom";
                    };
                    "Right Thumbstick Button" =                     {
                        kind = 1;
                        name = "Right Thumbstick Button";
                        nameLocalizationKey = "BUTTON_RIGHT_THUMBSTICK";
                        remappingKey = 21;
                        sfSymbolsName = "r.joystick.press.down";
                    };
                    "Right Trigger" =                     {
                        kind = 1;
                        name = "Right Trigger";
                        nameLocalizationKey = "RIGHT_TRIGGER";
                        remappingKey = 19;
                        sfSymbolsName = "r2.rectangle.roundedtop";
                    };
                };
                dpads =                 {
                    "Direction Pad" =                     {
                        kind = 2;
                        name = "Direction Pad";
                        nameLocalizationKey = "DIRECTION_PAD";
                        remappingKey = 0;
                        sfSymbolsName = dpad;
                    };
                    "Left Thumbstick" =                     {
                        kind = 2;
                        name = "Left Thumbstick";
                        nameLocalizationKey = "LEFT_THUMBSTICK";
                        remappingKey = 10;
                        sfSymbolsName = "l.joystick";
                    };
                    "Right Thumbstick" =                     {
                        kind = 2;
                        name = "Right Thumbstick";
                        nameLocalizationKey = "RIGHT_THUMBSTICK";
                        remappingKey = 14;
                        sfSymbolsName = "r.joystick";
                    };
                };
                hidden = 0;
                logoSfSymbolsName = "gamecontroller.fill";
                modifiedDate = "2024-05-08 18:25:35 +0900";
                name = "Logicool Dual Action";
                persistentIdentifier = "LOGICAL_DEVICE(9C6D26AF)";
                productCategoryKey = "PRODUCT_CATEGORY_HID";
                supportsHaptics = 0;
                supportsLight = 0;
            }
        );
        tombstones =         (
        );
    };
    games =     {
        data =         (
                        {
                bundleIdentifier = "com.apple.default";
                controllerToCompatibilityModeMappings =                 {
                };
                controllerToProfileMappings =                 {
                    "LOGICAL_DEVICE(9C6D26AF)" = "42381781-1B0C-467E-AA7D-823C4CE359D1";
                };
                modifiedDate = "2024-05-08 11:39:52 +0900";
                title = "\\U3059\\U3079\\U3066\\U306e\\U30b2\\U30fc\\U30e0";
            }
        );
        tombstones =         (
        );
    };
    settingsVersion = "10.1.21";
    showGCPrefsPane = 1;
}
```
</details>

# Unity Input System 準備

メニューのウィンドウからパッケージマネージャーを表示します。

![article010_03.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/e241ac18-3d00-98f6-1299-79e38b397741.png)

Unity レジストリからInput Systemを選択します。右側にインストールボタンがあるので押します。以前のUnity 2021ではインストールボタンは右下でした。

# 素のままのInput SystemでF310を使ってみる

何もせずともゲームパッドは使えます。しかし、Input Systemはジョイスティックとして認識してしまい、左スティックくらいしかまともに使えません。

どうにかならないものかとウェブを検索すると、Unity Communityにズバリの[スレッド](https://forum.unity.com/threads/controller-wrongly-characterized-as-joystick-for-input-logitech-dual-action.879454/)がありました。スレッドを見ていくとC#ファイルがあるので、ダウンロードしUnityのプロジェクトのAssetsに放り込んであげればいいはずです。ですがジョイスティックのまま何も変わりませんでした。

とりあえず、このソースコードを起点に修正していきます。

# Input Debuggerで確認

Input Systemをインストールすると、メニューからウィンドウ→分析→Input Debuggerでウィンドウを開くことができるようになります。

![article010_04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/156f6c62-10ce-6910-232f-6c67c4d1350e.png)

Devicesに **Logicool Logicool Dual Action** を確認できます。これはシステム情報にあったUSB HIDの文字列です。

ツリーのLayoutsを開いていくと、Joysticksの下に **Logicool Dual Action** にいます。本来のGamepadsの下にいてほしいのですが…

Devicesの **Logicool Logicool Dual Action** をダブルクリックか、副ボタンクリックのメニューで **Device Debug View** のウィンドウを開くことができます。

![article010_05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/50bd9104-f1f8-f802-ec3b-c92f78bbf93d.png)

ウィンドウ右に **HID Descriptor** と **State** というボタンがあって、押すとさらにウィンドウが開きます。これらのウィンドウを並べてゲームパッドの状態を確認しながらコーディングしていきます。

# オリジナル・ソースコードの修正

Input Systemの[ドキュメント](https://docs.unity3d.com/ja/Packages/com.unity.inputsystem@1.4/manual/HID.html)をあわせて見ながらチェックします。

:::note info
日本語に翻訳されたドキュメントのバージョンは1.4.3でした。
もっと新しいバージョンのページもありますが、今使う範囲では1.4.3で十分でした。
:::

前記スレッドのソースコードを抜粋します。

`WithManufacturer` と `WithProduct` に注目すると、Logitechとなっています。

```csharp:LogitechDualActionHID.cs
public class LogitechDualActionHID : Gamepad
{
    static LogitechDualActionHID()
    {
        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)", 
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithManufacturer("Logitech")
                .WithProduct("Logitech Dual Action"));
    }

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    static void Init() { }
}
```

実はLogicoolは日本法人の社名で、本当はスイスに本社を置くLogitechがグローバル名称です。商標の都合でこの様になっています。なんと日本で販売されているF310はHID Descriptorまで日本対応してあったんですね。

![article010_06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/c0e5ed5f-d0ae-7f1a-ee82-d8afc644deea.png)

そこで以下のように変更します。

```csharp:LogitechDualActionHID.cs
public class LogitechDualActionHID : Gamepad
{
    static LogitechDualActionHID()
    {
        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithManufacturer("Logitech")
                .WithProduct("Logicech Dual Action"));

        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithManufacturer("Logicool")
                .WithProduct("Logicool Dual Action"));

        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithCapability("vendorId", 0x46D) // Logitech Inc.
                .WithCapability("productId", 0xC216)); // F310 Gamepad [DirectInput Mode]
    }

    [RuntimeInitializeOnLoadMethod]
    static void Init() { }
}
```

LogitechとLogicoolの両方に対応できるように登録します。

**HID Descriptor** ウィンドウでHIDのVender IDとProduct IDを確認できるので、万が一まったく違う未知の名称だったとしても動作するようにIDでも登録しておきます。

次にボタンの定義をしている LogitechDualActionInputReport2.cs も見ていきます。

```csharp:LogitechDualActionInputReport2.cs
[StructLayout(LayoutKind.Explicit, Size = 32)]
struct LogitechDualActionInputReport2 : IInputStateTypeInfo
{
```

HID DescriptorではInput Report Sizeが8です。 `Size = 8` に修正します。Output Reportの7バイトですが、ツリーの40 Elementsを展開するとInputのあとに順に並んでいるだけなので無視して構いません。ゲームパッドによっては、Input・Output・Fautureのビット順がバラバラのときがあるので、そのときは適切なサイズにします。 **Device Debug View** では48bits(== 6 bytes)になってます。ゲームパッドの定義が完成すると、ここが64bits (== 8 bytes)になります。

:::note info
PS4 DualShock は、パケットの終端近くの30バイト目にバッテリーレベルがあるようです。
:::

スティクの上下方向で、下に倒すと-1で上に倒すと+1になってほしいのですが反転しています。左右スティックとも反転していました。ここは修正します。

```csharp:LogitechDualActionInputReport2.cs
    [InputControl(name = "leftStick/y", offset = 1, defaultState = 127, format = "BYTE", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5")]
    [InputControl(name = "leftStick/down", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=-1,clampMax=0,invert")]
    [InputControl(name = "leftStick/up", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=0,clampMax=1")]
```

左右スティックを押し込んだときのボタン定義がされていません。6バイト目のビット6とビット7だったので追加してあげます。

F310にはMODEボタンがあります。DPADと左スティックを入れ替える機能です。こちらは7バイト目のビット3が対応します。これも追加します。

# 修正後のソースコード

2つのソースコードを、UnityのプロジェクトのAssets以下の適当な場所に置きます。自動的に実行されます。

```csharp:LogitechDualActionHID.cs
using UnityEditor;
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.Layouts;

[InputControlLayout(stateType = typeof(LogitechDualActionInputReport))]
#if UNITY_EDITOR
[InitializeOnLoad] // Make sure static constructor is called during startup
#endif
public class LogitechDualActionHID : Gamepad
{
    static LogitechDualActionHID()
    {
        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithManufacturer("Logitech")
                .WithProduct("Logicech Dual Action"));

        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithManufacturer("Logicool")
                .WithProduct("Logicool Dual Action"));

        InputSystem.RegisterLayout<LogitechDualActionHID>("Logitech Dual Action (Unofficial)",
            new InputDeviceMatcher()
                .WithInterface("HID")
                .WithCapability("vendorId", 0x46D) // Logitech Inc.
                .WithCapability("productId", 0xC216)); // F310 Gamepad [DirectInput Mode]
    }

    [RuntimeInitializeOnLoadMethod]
    static void Init() { }
}
```

```csharp:LogitechDualActionInputReport.cs
using System.Runtime.InteropServices;
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.LowLevel;
using UnityEngine.InputSystem.Utilities;
using UnityEngine.InputSystem.Layouts;

[StructLayout(LayoutKind.Explicit, Size = 8)]
struct LogitechDualActionInputReport : IInputStateTypeInfo
{
    public FourCC format => new FourCC('H', 'I', 'D');

    [InputControl(name = "leftStick", format = "VC2B", layout = "Stick", displayName = "Left Stick", offset = 0)]
    [InputControl(name = "leftStick/x", defaultState = 127, format = "BYTE", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5")]
    [InputControl(name = "leftStick/left", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=-1,clampMax=0,invert")]
    [InputControl(name = "leftStick/right", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=0,clampMax=1")]
    [InputControl(name = "leftStick/y", offset = 1, defaultState = 127, format = "BYTE", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,invert")]
    [InputControl(name = "leftStick/down", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=0,clampMax=1")]
    [InputControl(name = "leftStick/up", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=-1,clampMax=0,invert")]
    [FieldOffset(0)] public byte leftStickX;
    [FieldOffset(1)] public byte leftStickY;

    [InputControl(name = "rightStick", format = "VC2B", layout = "Stick", displayName = "Right Stick", offset = 2)]
    [InputControl(name = "rightStick/x", defaultState = 127, format = "BYTE", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5")]
    [InputControl(name = "rightStick/left", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=-1,clampMax=0,invert")]
    [InputControl(name = "rightStick/right", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=0,clampMax=1")]
    [InputControl(name = "rightStick/y", offset = 1, defaultState = 127, format = "BYTE", parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,invert")]
    [InputControl(name = "rightStick/up", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=0,clampMax=1")]
    [InputControl(name = "rightStick/down", offset = 1, parameters = "normalize,normalizeMin=0,normalizeMax=1,normalizeZero=0.5,clamp=2,clampMin=-1,clampMax=0,invert")]
    [FieldOffset(2)] public byte rightStickX;
    [FieldOffset(3)] public byte rightStickY;

    [InputControl(name = "dpad", displayName = "D-Pad", format = "BIT", layout = "Dpad", sizeInBits = 4, defaultState = 8)]
    [InputControl(name = "dpad/up", displayName = "D-Pad Down", format = "BIT", layout = "DiscreteButton", parameters = "minValue=7,maxValue=1,nullValue=8,wrapAtValue=7", bit = 0, sizeInBits = 4)]
    [InputControl(name = "dpad/right", displayName = "D-Pad Left", format = "BIT", layout = "DiscreteButton", parameters = "minValue=1,maxValue=3", bit = 0, sizeInBits = 4)]
    [InputControl(name = "dpad/down", displayName = "D-Pad Right", format = "BIT", layout = "DiscreteButton", parameters = "minValue=3,maxValue=5", bit = 0, sizeInBits = 4)]
    [InputControl(name = "dpad/left", displayName = "D-Pad Up", format = "BIT", layout = "DiscreteButton", parameters = "minValue=5, maxValue=7", bit = 0, sizeInBits = 4)]
    [InputControl(name = "buttonWest", displayName = "X", layout = "Button", format = "BIT", bit = 4)]
    [InputControl(name = "buttonSouth", displayName = "A", layout = "Button", format = "BIT", bit = 5)]
    [InputControl(name = "buttonEast", displayName = "B", layout = "Button", format = "BIT", bit = 6)]
    [InputControl(name = "buttonNorth", displayName = "Y", layout = "Button", format = "BIT", bit = 7)]
    [FieldOffset(4)] public byte buttons1;

    [InputControl(name = "leftShoulder", displayName = "Left Bumper", layout = "Button", format = "BIT", offset = 5, bit = 0)]
    [InputControl(name = "rightShoulder", displayName = "Right Bumper", layout = "Button", format = "BIT", offset = 5, bit = 1)]
    [InputControl(name = "leftTrigger", displayName = "Left Trigger", layout = "Button", format = "BIT", offset = 5, bit = 2)]
    [InputControl(name = "rightTrigger", displayName = "Right Trigger", layout = "Button", format = "BIT", offset = 5, bit = 3)]
    [InputControl(name = "back", displayName = "Back", layout = "Button", format = "BIT", offset = 5, bit = 4)]
    [InputControl(name = "start", displayName = "Start", layout = "Button", format = "BIT", offset = 5, bit = 5)]
    [InputControl(name = "leftStickPress", displayName = "Left Stick Press", layout = "Button", format = "BIT", offset = 5, bit = 6)]
    [InputControl(name = "rightStickPress", displayName = "Right Stick Press", layout = "Button", format = "BIT", offset = 5, bit = 7)]
    [FieldOffset(5)] public byte buttons2;

    [InputControl(name = "mode", displayName = "Mode", layout = "Button", format = "BIT", offset = 6, bit = 3)]
    [FieldOffset(6)] public byte buttons3;

    [InputControl(name = "reserve1", layout = "Button", format = "BYTE", offset = 7, bit = 0)]
    [FieldOffset(7)] public byte reserve1;
}
```

# スクリプト配置後の様子

**HID Descriptor** はこのようになりました。ゲームパッドの各種ボタンを操作すると、期待通りの動作していることがわかると思います。

![article010_07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/f8953c4d-c7d7-d617-2c4b-801b14cbf14e.png)

**Input Debug** はこのようになりました。JoysticksからGamepadsに移動しています。Matching Devicesが3つ並んでいます。

![article010_08.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/2c8a28d5-14a0-2f4d-3b60-505b9d90afd3.png)

プロジェクトでInput Actionsを新規作成し、GameObjectのコンポーネントにPlayer Inputなど追加して動作を確認してみてください。スティックもボタンも思い通りに動くようになっていると思います。

:::note warn
なお、ゲームパッド中央のLogicoolボタンはDirectInputインターフェイスモードでは使用できません。HIDのパケットは届くのに、どのビットも変化しませんでした。そもそもマニュアルにXInputのみとあるので正しい動作だと思います。
:::

# さいごに

時間をかけていろいろやったあとに、すでにQiitaの先輩が記事にされていることに気が付きました。

あ゛あ゛あ゛:sob:


https://qiita.com/marbledNEET/items/02dc9cfdede8183b17b4

