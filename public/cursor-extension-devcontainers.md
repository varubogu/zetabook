---
title: CursorでMS製DevContainerがインストールできなくなったので代替方法を探す
tags:
- cursor
- devcontainer
- ms-devcontainer
private: false
updated_at: ''
id: ''
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

いつからだか忘れましたがVS Code以外のVS Code派生エディター（Cursor、Windsurfなど）でMS製の拡張機能がインストールできなくなりました。
Cursorの拡張機能はCursorの「拡張機能」バーからインストールできますが、MS製の拡張機能はMarketplaceに表示されなくなっています。
昔インストールしたものはそのまま使えますが、今後のアップデートはおそらくできないでしょう。

今回はCursorでDevContainerを使うための代替方法を探してみました。

なおCursorではなくGitHubCopilot、Cline、Gemini CLI、Claude Codeなどを使ったりなどするならVS Codeを使うのが一番早いのでそちらを使いましょう。

## 先に結論

VS Codeでリポジトリの開発コンテナーを作って開きっぱなしにし、Cursor＋Anysphere製DevContainer拡張機能でそのコンテナーにアタッチするという使い方が一番現実的そうな気がします。

## 代替方法1: Anysphere製のDevContainer拡張機能を使う

Cursorの拡張機能バーから「DevContainer」で検索すると、Anysphere製のDevContainer拡張機能が見つかります。

![拡張機能検索結果](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/432981/79b9380c-694c-49f9-891c-ec92287fa9a5.png)

しかしMS製のDevContainer拡張機能と比べて機能が制限されているため、完全な代替とは言えません。

できること

- 作成済みのコンテナーに対するアタッチ（Cursorの「リモートエクスプローラー」などから）
- 今のフォルダーをコンテナーで開く
- コンテナーのビルド、リビルド（ただしdevcontainer.jsonは手動で作成する必要がある）

できないこと

- devcontainer.jsonの作成
- リポジトリをコンテナーにクローン＋ビルド（MS製DevContainerでいうところの「コンテナーボリュームにリポジトリを複製」コマンドに相当）
- Cursor上での新しい開発コンテナーの作成（コマンドパレット、リモートエクスプローラーからの操作）
- Cursorを閉じた時に自動的に開発コンテナーを終了する

つまり、Dev Container自体は使用できますが、初回のリポジトリクローンの際に初期設定する時に手順が増えて面倒になったという感じです。
具体的にいうと

1. devcontainer.jsonを手動で作成する必要がある
従来：VS Code上でコンテナーボリュームにリポジトリを複製し、devcontainer.jsonを自動生成してくれた。（ローカルにはクローンせずボリュームにクローン）
現在：手動でdevcontainer.json作成、コンテナビルド、その後リポジトリをコンテナーにクローンする必要がある。

2. DevContainerにアタッチする際、コンテナーが起動済みである必要がある
従来：VS CodeのDevContainer拡張機能でアクセスするだけで自動的に起動・終了してくれる
現在：なんらかの方法（Dockerから直接コンテナーを起動したり、Dev Container CLIなど）でコンテナーを起動し、Cursorでアタッチする必要があり、Cursorを閉じた時に自動的に終了しない。

これらを考慮すると、VS Codeでリポジトリの開発コンテナーを作って開きっぱなしにして、Cursorでそのコンテナーにアタッチするという使い方が一番現実的です。
ただし2つ立ち上げる分メモリは爆食いする点に注意。RAMが少ないPCは負荷があがって遅くなるなどもありえます。

## 代替方法2： Cursorを諦めてVS Codeを使う

元も子もないですが、直接VS Codeを使うのが一番早いです。
CursorだとDevContainer利用時に重くなるという意味でも実は一番まともかもしれません。

## まとめ

CursorでMS製DevContainer拡張機能が使えなくなったため、Anysphere製のDevContainer拡張機能を使うか、VS Codeを直接使うという方法があります。
Anysphere製のDevContainer拡張機能は機能が制限されているため、従来と同じ使い勝手にはならないでしょう。
~~どうやらこれがMicrosoftのやり方らしい。~~
