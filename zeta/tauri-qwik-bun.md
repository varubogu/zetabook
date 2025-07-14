---
title: 'Tauri(Rust)、Qwik、Bunを使って爆速なクロスプラットフォームアプリ（デスクトップ、スマホ、Web）を作る（環境構築編）'
emoji: 'rust'
type: tech
topics: ['Tauri', 'Rust', 'Qwik', 'TypeScript']
qiita_id: 'bee474d9e7e3da4ca69d'
published: true
---

## はじめに

今回はTauri、Qwik、Bunを使って爆速なクロスプラットフォームアプリ（デスクトップ、スマホ、Web）を作るための環境構築方法を紹介する。

## 今回の技術スタック

### Rustとは

ブラウザのFirefoxを開発しているMozillaが開発したプログラミング言語。
所有権システムなど独特な方法でメモリ安全性を保証しつつ処理を高速化するための機能が備わっている。
バックエンドとしてLLVMを使っているため、さまざまなプラットフォームに対応でき、ビルド時に高度な最適化が可能である。

### TypeScriptとは

JavaScriptの構文を拡張し、より高度な記述ができるようになった言語。
型システムが導入されているため、コードの品質を向上させることができる。
実際に使う際にはTypeScriptを直接解釈するのではなく、JavaScriptに変換された後にJavaScriptエンジンで実行される。

### Tauriとは

デスクトップアプリのフレームワーク。
特徴としてはUIはWebの技術（HTML/CSS/JavaScript）で書くことができ、ロジックはRustなどで書くことができる。
UIのイベントもJavaScriptで受けとり、ロジック担当のRustへの受け渡しができる。
フロントエンドがWeb技術なのでReactなどのWebアプリの資産を活用できる他、プラットフォームごとのUIの違いが発生しにくくなる。

似たような構成のアプリとしてはVS Codeなどで使われるElectronがある。
あちらはChromiumブラウザを丸ごと含んだ構成となるためサイズが大きくなりやすい。
TauriはOSに含まれるWebViewを使って描画するため、Electronよりもバイナリ、実行時メモリなどが軽量になる。
それでいてロジックはRustベースなので高度な最適化が可能である。

最近Tauri 2.0がリリースされ、スマホアプリにも対応した。

### Qwikとは

WebアプリやWebサイト開発のためのフレームワーク。
おそらく現状存在するJavaScriptベースのWebフレームワークの中でもっとも高速なのではないかと思っている。

jsxやtsxを使って書かれるが、Reactや他のフレームワークとは異なる点として必要になるまで読み込まない、細かい単位に分割されて必要になった部分だけをロードするなどの特徴がある。
そのため初期表示までの速度が非常に早く、ユーザー体験を向上させることができる。

### Bunとは

サーバーサイドで動くJavaScript環境としてNode.jsがある。
Node.jsはV8エンジン（Chromeなどで使われている）を使ってJavaScriptを実行するため、ブラウザと同じようにJavaScriptを使ってサーバーサイドの処理を書くことができる。

BunはNode.jsと互換性があり、以下の特徴を持つ：

- JavaScriptランタイムとして最速レベルのパフォーマンス
- 高速なパッケージマネージャー機能
- Node.js互換APIの提供
- ビルトインのバンドラー機能
- テストランナーの内蔵

## 環境構築手順

### 必要なものをインストール

- Rust
- OSに応じたRustのビルドツール（WindowsならMSVC、macOSならXcodeなど）
- Bun（npm互換のあるツールならなんでも良い）
- VS Code、WebStorm、RustRoverなど好きなエディター/IDE

### プロジェクトの作成

#### Qwikのプロジェクトを作成

まずベースとなるQwikのプロジェクトを作成する。
対話式で進んでいくので、今回は以下のように設定した。

```bash
bun create qwik@latest

# 作成したいディレクトリに既にいる状態なのでカレントディレクトリを指定。
# 名称を指定すればそのディレクトリが作成される。
◇  Where would you like to create your new project? (Use '.' or './' for current directory)
│  ./

# プロジェクトの種類を決める。今回は空のアプリとして作成した。
Select a starter
│  ● Empty App (Qwik City + Qwik) (Blank project with routing included)
│  ○ Library (Qwik)
│  ○ Playground App (Qwik City + Qwik)

# Bunの依存関係をインストール
◇  Would you like to install bun dependencies?
│  Yes

# リポジトリ作成済みだったので今回はNoを選択
◇  Initialize a new git repository?
│  No

◇  App Created 🐰
```

いったんこの時点でQwikの実行をしてみる

```bash
bun dev
```

うん？エラーが出た。
よく見たら`bun install`が失敗したので手動でやってくれと言われていた。

```bash
■   bun install failed
│   You might need to run "bun install" manually inside the root of the project.
```

というわけで、`bun install`を実行してから再度実行する。

```bash
bun install
bun dev
```

<macro>
zenn: "![Qwik Hello](/images/tauri-qwik-bun/qwik-hello.png)"
qiita: "(![Qwik Hello](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/432981/9819c968-eeb8-e39f-1c56-78953e4522d0.png))"
</macro>

うまくいった。

#### QwikにTauriを組み込む

まずTauriのCLIをインストールする。
ビルド時にしか使わないので`-D`オプションをつけて開発の依存関係としてインストールする。

```bash
bun add -D @tauri-apps/cli
```

```json package.json
{
  "devDependencies": {
    // ↓追加される
    "@tauri-apps/cli": "^1.0.0-beta.1"
  }
}
```

Bunからコマンドを使えるよう、`package.json`にスクリプトを追加する。
ついでにアプリ名を`qwik-app`にしておく。（Tauriの初期化時のアプリ名初期表示になるので）

```json package.json
{
  // ↓アプリ名変更
  "name": "qwik-app",
  ...
  "scripts": {
    // ↓追加
    "tauri": "tauri"
  }
}
```

ここまでやってTauri CLIで初期化を実行する

```bash
bun tauri init
```

```bash
✔ What is your app name? · qwik-app
✔ What should the window title be? · qwik-app
✔ Where are your web assets (HTML/CSS/JS) located, relative to the "<current dir>/src-tauri/tauri.conf.json" file that will be created? · ../dist
✔ What is the url of your dev server? ·
✔ What is your frontend dev command? · bun dev
✔ What is your frontend build command? · bun build
```

これが終わると`src-tauri`ディレクトリが作成され、その中にTauriの設定ファイルが`src-tauri/tauri.conf.json`が作成される。

```json src-tauri/tauri.conf.json
{
  "$schema": "../node_modules/@tauri-apps/cli/config.schema.json",
  // ↓アプリ名変更
  "productName": "qwik-app",
  "version": "0.1.0",
  // ↓アプリ識別子変更
  "identifier": "com.qwik-app.app",
  // ↓ビルド時の設定追加
  "build": {
    "frontendDist": "../dist",
    "beforeDevCommand": "bun run dev",
    "beforeBuildCommand": "bun run build",
    "devUrl": "http://localhost:5173"

  },
}
```

この設定ファイルのうち、`identifier`はアプリの一意な識別子である。
デフォルトだと`com.tauri.app`になっているが、これを変更しないとビルドができない。
devUrlもデフォルトだと違うホスト名・ポートになっていたので、Qwikの設定に合わせて変更した。

#### Tauriのビルド

まずHTML部分は静的サイトとして用意する必要があるため、qwikプロジェクトをビルドする。
が、その前にビルド設定を少しいじる必要がある。
具体的にいうとTauriとの接続用にホスト名とポートを指定する必要がある。

```ts vite.config.ts
// ↓ホスト定義
const host = process.env.TAURI_ENV_HOST || "localhost";
export default defineConfig((): UserConfig => {
  return {
    // ↓追加
    server: {
      headers: {
        // Don't cache the server response in dev mode
        "Cache-Control": "public, max-age=0",
      },
      strictPort: true,
      host: host || false,
      port: 5173,
    },
  }
});
```

改めてQwikをビルドする

```bash
bun qwik build
```

出力結果はデフォルトだと`dist`ディレクトリに出力される。
このdistディレクトリはTauriの設定ファイルで指定した`frontendDist`に対応する必要がある。

Tauriの設定も変更したか再確認。
一応生ソースを貼っておく

```json src-tauri/tauri.conf.json
{
  "$schema": "../node_modules/@tauri-apps/cli/config.schema.json",
  "productName": "qwik-app",
  "version": "0.1.0",
  "identifier": "com.qwik-app.dev",
  "build": {
    "frontendDist": "../dist",
    "beforeDevCommand": "bun run dev",
    "beforeBuildCommand": "bun run build",
    "devUrl": "http://localhost:5173"
  },
  "app": {
    "windows": [
      {
        "title": "qwik-app",
        "width": 800,
        "height": 600,
        "resizable": true,
        "fullscreen": false
      }
    ],
    "security": {
      "csp": null
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

あとはTauriのビルドと実行を行う。

```bash
bun tauri dev
```

<macro>
zenn: "![Qwik Hello](/images/qwik-hello/tauri-qwik-hello.png)"
qiita: "(![Qwik Hello](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/432981/83326967-3f9d-3403-deec-d1d7e104657e.png))"
</macro>

## おわりに

ここまででHello Worldレベルの実装が完了した。
まだVS Code上でのデバッグやビルドの設定などは行っていないが、それはまた別の機会に。
