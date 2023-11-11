---
title: Unityのバージョン変更が動かなかったときにしたこと
tags:
  - macOS
  - Unity
private: false
updated_at: '2023-11-11T10:27:56+09:00'
id: e54c7918b0182ed8adfb
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

macOSでUnityを使っています。マイナーバージョンアップは頻繁にあるので、ときどきまとめて追随させてきました。ときには2020から2021へみたいにメジャーバージョンを変えてみたりもしています。

ちょっと前までUnity Hubからバージョン変更できていたのですが、いつの間にか変換中に固まるようになりました。そこで手動でファイルを編集して強制アップデートしたので、そのときの方法を記録します。

# プロジェクトツリー確認

Unityプロジェクトに必須のフォルダは以下になります。

* Assets
* Packages
* ProjectSettings

以下はUnity Editorが生成するので消しても構いません。
* Library
* Logs
* UserSettings
* *.csproj
* *.sln
* その他

# 事前作業

パッケージマネージャー画面で、なるべく更新しておいたほうがよいかと思います。

# バージョン変更

使用中のEditorのバージョンは、`./ProjectSettings/ProjectVersion.txt`にテキストファイルとして記載されています。2行しかありません。

```
m_EditorVersion: 2021.3.25f1
m_EditorVersionWithRevision: 2021.3.25f1 (68ef2c4f8861)
```

バージョンアップしたいエディタの`ProjectVersion.txt`が欲しいので、新規にプロジェクトをひとつ作ります。2021.3.32f1はこうでした。

```
m_EditorVersion: 2021.3.32f1
m_EditorVersionWithRevision: 2021.3.32f1 (3b9dae9532f5)
```

メジャーバージョンを変えるときも基本おなじです。一応マイナーバージョンを最新にしてから、メジャーバージョンを１つずつ変えて、なにかおかしくなっていないかよく調べたほうがよいと思います。

2022.3.12f1の`ProjectVersion.txt`はこうです。

```
m_EditorVersion: 2022.3.12f1
m_EditorVersionWithRevision: 2022.3.12f1 (4fe6e059c7ef)
```

ファイルを書き換えたら、Unity Hubのバージョンを変えてEditorを立ち上げます。もちろんUnity Hubを使ってない人は、アプリから直接読み込みます。

# パッケージのエラー

時々パッケージのバージョンでエラーがでます。Editor内蔵パッケージのため、ダウンロードして更新はできません。こちらも手直ししていきます。

パッケージは、`./Packages/manifest.json`にかかれています。パッケージマネージャーでワーニングアイコンがでていたところをメモし、バージョンを書き換えます。

私の場合、2021.3.32f1にしたときに書き換えた行だけ抜き出しています。

```
    "com.unity.render-pipelines.high-definition": "12.1.13",
    "com.unity.render-pipelines.universal": "12.1.13",
    "com.unity.shadergraph": "12.1.13",
```

次に2022.3.12f1にしたときも手修正しました。

```
    "com.unity.render-pipelines.high-definition": "14.0.9",
    "com.unity.render-pipelines.universal": "14.0.9",
    "com.unity.shadergraph": "14.0.9",
```

以上で無事問題なく新しいバージョンで動くようになりました。

# おわりに

アセットが少なかったのでうまくいきました。規模が大きいプロジェクトですと、こんなにうまくはいかないような気がします。

Unityのほうで対応してほしいです。お願いします。
