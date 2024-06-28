---
title: 'Windows環境でZedをビルドして使う'
emoji: '✏'
type: tech
topics: ["Zod", "Windows"]
qiita_id: ''
published: true
---

## はじめに

最近話題にあがるようになった軽量エディタであるZedは、現在Mac版のみの提供である。
しかし公式には提供されていないもののオープンソースかつクロスプラットフォーム対応なので自分でビルドすればWindowsやLinuxでも使うことができる。

公式にWindows向けのビルド方法が書いてあるのでそれを見ながらやっていく。
<https://github.com/zed-industries/zed/blob/main/docs/src/development/windows.md>

今回はインストーラーを作成する形式ではなくexeを直接生成するやり方である。
また、この流れで発生するいかなる事象について、私は一切の責任を負えないことを了承のうえで実行してほしい。

## 環境

- OS: Windows11 x64
- Rust: 1.79.0
- Visual Studio 2022 Community Edition

## セットアップ

### Rustのインストール（アップデート）

ZedはRustで記述されているため、まずはRustのインストールを行う。
今回はWindows向けのビルドなのでWSL環境やDevContainer環境でのRustが使えないことは注意が必要だ。

公式からダウンロード＆インストールしよう。
<https://www.rust-lang.org/ja/tools/install>

パッケージ管理ツールを使っている方はそちらからインストールでも良い。
例えばwingetを使っているなら以下のコマンドを打つ。

```ps1
winget install Rustlang.Rustup
```

アップデートの場合は下記コマンドを実行するだけで良い。

```ps1
rustup update
```

### MSVCのインストール

次にWindows向けビルドをするためにMSVCが必要になる。
主に2つのやり方があるのでどちらも紹介する。

#### Visual Studioを使う場合

Visual Studio Installerの画面で追加コンポーネントを選ぶことができる。
このうち必要になるのは「C++によるデスクトップ開発」の項目だ。
「MSVC」が必要になるのでそれを選んでインストールしよう。
![Visual Studio Installer](/images/zed-windows-build/image.png)

#### Build Tools for Visual Studioを使う場合

VisualStudioを入れる余裕がない人の選択肢がこちら。

下記からインストールできる
<https://visualstudio.microsoft.com/ja/visual-cpp-build-tools/>

一応winget版も書いておく。

```ps1
winget install Microsoft.VisualStudio.2022.BuildTools
```

### wasmツールチェインのインストール

下記のコマンドでインストールしよう。

```ps1
rustup target add wasm32-wasi
```

### リポジトリをCloneする

```ps1
git clone https://github.com/zed-industries/zed.git
```

### rust-toolchain.tomlにツールチェインを記載

リポジトリのルートにあるファイルを編集して下記のように書き換える。
targetsの中身をMSVCのものに変えれば良い。

```Cargo.toml
[toolchain]
channel = "1.79"
profile = "minimal"
components = [ "rustfmt", "clippy" ]
targets = [ "stable-x86_64-pc-windows-msvc" ]
```

### Cargoでビルドする

以下のコマンドでビルドを行う。
長時間かかるしメモリ使用率・CPU使用率も跳ね上がるので事前に他のアプリは終了させておこう。
~~自分はというとRustRover内のコンソールでビルドしてしまってえらいことになった。デバッグビルドはともかくリリースビルドする時は落としておきたい~~

```ps1
cargo build --release
```

ビルド終了後、リポジトリのある位置から `target/release/` といけば本体のzod.exeファイルがある。
これを好きな位置に配置して実行しよう。

![zod.exe](/images/zed-windows-build/image-1.png)

これで完了。

## Zodの感想

まずビルドしての感想。
Windowsビルドの難易度は全然高くないと思った。
英語を読む力（翻訳系アプリを使っても良い）がある程度あってWindowsアプリを何かしら作ったことがある人であれば問題なくできるだろう。
ドキュメントが簡潔にまとまってるのでかなり良いなという印象。

次にZed自体の感想。
起動時間がマジで早い。
サクラエディタより若干遅いが1秒かからず起動。
VSCodeより起動が早いのでササッとコード確認したいときには便利。

## 余談：サクラエディタとメモリ使用量を比較してみた

Zod 174.1MB
<https://x.com/varubogu_tysk/status/1806462060324425980>

サクラエディタ 3.4MB
<https://x.com/varubogu_tysk/status/1806435538326749320>

あれ、もしかしてサクラエディタすごい・・・？
