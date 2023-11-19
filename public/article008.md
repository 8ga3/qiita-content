---
title: Qtでポイントクラウド
tags:
  - Qt
  - 3D
  - Shader
  - PointCloud
private: false
updated_at: ''
id: 
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

Qtは組み込みシステムからOSSと様々なシーンで採用されています。それでも3DCGを扱えることを知る人は、Qtでプログラミングしてみた人以外はあまり知られていないと思います。

以前、Qtで3次元データのポイントクラウド（点群）を表示する方法を調べ、実装例をGitHubに公開していました。

下記の４つの環境で、それぞれ実装例を作ってます。

* Qt Quick 3D 6.6
* Qt Quick 3D 5.15.2
* Qt 3D 5.15.2
* Qt Quick 5.15.2 + OpenGL

この記事でまとめて紹介していきます。情報が少なく苦労したものもあります。誰かの役に立てば幸いです。

なお普段使っているmacOS環境ベースで書いています。**Windows 11で動作しましたが確認したのは１年くらい前です。2023年11月時点での最新版の動作確認はしていません。**

# Qt Quick 3D 6.6 環境のポイントクラウド

Qt 6のQt Quick 3Dを使って、ポイントサイズを変更するサンプルアプリです。


https://github.com/8ga3/PointSize_Qt6_Quick3D


![pc_qtquick3d6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/7a39c581-8600-2ce0-acfa-8f603d79a44e.png)

## Features
あとに記載するQt5では、QMLだけでポイントクラウドのポイントサイズを変えられませんでした。Qt6からQuick 3Dの`DefaultMaterial`のプロパティ`pointSize`を変更するだけで、ポイントの大きさを表示が変えられるようになりました。 ポイントの形状は正方形になります。

```qml
Model {
    id: cube
    geometry: PointCloudCubeGeometry {
        pointCount: 10000
    }
    materials: [
        DefaultMaterial {
            lighting: DefaultMaterial.NoLighting
            pointSize: Screen.devicePixelRatio * 5
            vertexColorsEnabled: true
        }
    ]
}
```

`Screen.devicePixelRatio` をかけることで、普通のディスプレイと高画素密度のディスプレイの双方同じ見え方になります。

## Requirement
* Qt Creator
* CMake
* Qt 6.6.0

# Qt Quick 3D 5.15.2 環境のポイントクラウド

Qt 5.15.2のQt Quick 3Dを使って、ポイントサイズを擬似的に変更するサンプルアプリです。


https://github.com/8ga3/PointSize_Qt5_Quick3D


![pc_qtquick3d5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/a3b1e9ac-6c7f-a84b-cdfb-2b4966fce6a3.png)

## Features
Qt5のQt Quick 3Dでは、PrimitiveTypeがポイントに設定した時のサイズを簡単に変更することができません。 Qt Quick 3Dのソースコードを調べてみると、シェーダーコンパイラにわたす前に`glPointSize`を削除しているようでした。そこでポイントの代わりにビルボードを作成し、Shaderでポイントサイズが可変したかのように見せています。やっていることはパーティクルと同じかと思います。

C++のコードでは、トライアングル２枚を生成しています。座標はすべて同じにしていて、Vertex shaderでポイントサイズにあわせて座標を変更しています。Fragment shaderで円を描画しています。お好みの形やテクスチャを貼るように書き換えてもいいと思います。

```cpp
    for (int i = 0; i < m_fPointCount; ++i) {
        float x = randomFloat(-5.0f, +5.0f);
        float y = randomFloat(-5.0f, +5.0f);
        float z = randomFloat(-5.0f, +5.0f);

        // vertex
        for (int j = 0; j < 4; ++j) {
            *p++ = x;
            *p++ = y;
            *p++ = z;
            *p++ = float(((j+1) >> 1) & 1); // Texture u
            *p++ = float(( j    >> 1) & 1); // Texture v
        }

        // index
        *idx++ = i * 4;
        *idx++ = i * 4 + 1;
        *idx++ = i * 4 + 2;

        *idx++ = i * 4;
        *idx++ = i * 4 + 2;
        *idx++ = i * 4 + 3;
    }
```

```glsl:ChangePoints.vert
in vec3 attr_pos;
in vec2 attr_uv0;
uniform mat4 modelViewProjection;
out vec3 pos;
out vec2 uv0;

void main() {
    float sx = sin(pitch);
    float cx = cos(pitch);
    mat3 mx = mat3(
        1.0, 0.0, 0.0,
        0.0,  cx, -sx,
        0.0,  sx,  cx);
    float sy = sin(yaw);
    float cy = cos(yaw);
    mat3 my = mat3(
         cy, 0.0,  sy,
        0.0, 1.0, 0.0,
        -sy, 0.0,  cy);

    pos = attr_pos;
    uv0 = attr_uv0;
    float x = (float((((gl_VertexID+1) / 2) % 2) * 2.0) - 1.0) * length;
    float y = (float((( gl_VertexID    / 2) % 2) * 2.0) - 1.0) * length;
    gl_Position = modelViewProjection * vec4(pos + mx * my * vec3(x, y, 0.0), 1.0);
}
```

## Requirement
* Qt Creator
* CMake
* Qt 5.15.2

## Note
通常のポイントサイズは、Perspective/Orthographic(透視投影/平行投影)のいずれでも大きさは一定です。 このサンプルアプリは擬似的にサイズを変更しているため、透視射影ではカメラからの距離によってサイズが変わってしまいます。

# Qt3D 5.15.2 環境のポイントクラウド

QtのQt3Dを使って、ポイントサイズを変更するサンプルアプリです。


https://github.com/8ga3/PointSize_Qt5_3D


![pc_qt3d5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/378d0cf9-48c0-b4bc-9c86-b8551c7d745f.png)

## Features
Qt5のQt3Dを用いて、ポイントクラウドのポイントサイズを変更する方法を示しています。 Qt6.xでも多少の変更で可能と考えられます。Qt3DはQt Quick 3Dと比べてより低レベルのためちょっと難しいですが、ほとんどをqmlで記述できました。

`PointSize`で大きさを設定しています。`Screen.devicePixelRatio` をかけることで、普通のディスプレイと高画素密度のディスプレイの双方同じ見え方になります。

```qml:stage.qml
...

    components: [
        RenderSettings {
            activeFrameGraph: RenderSurfaceSelector {
                ForwardRenderer {
                    clearColor: Qt.rgba(0.95, 1.0, 1.0, 1.0)
                    CameraSelector {
                        camera: root.perspective == true ? perspectiveCamera : orthographicCamera
                        RenderStateSet {
                            renderStates: [
                                PointSize {
                                    sizeMode: PointSize.Fixed
                                    // Common values are 1.0 on normal displays and 2.0 on Apple "retina" displays.
                                    value: root.pointSize * Screen.devicePixelRatio
                                },
                                DepthTest {
                                    depthFunction: DepthTest.Less
                                }
                            ]
                        }
                    }
                }
            }
        },
        InputSettings {}
    ]

...
```

## Requirement
* Qt Creator
* CMake
* Qt 5.15.2

# Qt 5.15.2 GLSE 環境のポイントクラウド

Qt 5.15.2のOpenGLを使って、ポイントサイズを変更するサンプルアプリです。


https://github.com/8ga3/PointSize_Qt5_GLSL


![pc_qtglsl.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/39665f26-02df-2c97-5e68-ee9a352a398d.png)

## Features

Qt5のOpenGLの`glPointSize()`を使って変更しています。`m_devicePixelRatio`をかけることで、普通のディスプレイと高画素密度のディスプレイの双方同じ見え方になります。

```cpp
  glPointSize(m_fPointSize * m_devicePixelRatio);
```

## Requirement
* Qt Creator
* qmake
* Qt 5.15.2

## Note

* `/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/System/Library/Frameworks/OpenGL.framework`
`xcode-select --install`でインストールしたSDKには`glPointSize`が定義されていました。

* `/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX13.1.sdk/System/Library/Frameworks/OpenGL.framework`
XcodeのからインストールするSDKにはglPointSizeがありませんでした。 環境設定、OSによってビルドに失敗するかもしれません。

* `/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX14.sdk/System/Library/Frameworks/OpenGL.framework`
まだ調べてません。

* `/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX15.sdk/System/Library/Frameworks/OpenGL.framework`
まだ調べてません。

# Build Settings for M1 mac

M1 macではarm64とx86_64を明示的に指定する必要があります。 ビルド設定のCMake > CMAKE_OSX_ARCHITECTURESにx86_64と記述します。 Qt 5.15.4以降はarm64にサポート外ながら一応対応していてUniversal Binaryになります。Qt 6.xを使っているのであれば、何も気にする必要はありません。

![qt5_cmake_build_settings.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569302/1b7e9f64-74ef-ed69-601a-dafd6b5367e1.png)


# おわりに

時間をかけて調べ上げましたが、色々あって使い道を失ってしまいました。ここの記事にすることで供養したいと思います。
