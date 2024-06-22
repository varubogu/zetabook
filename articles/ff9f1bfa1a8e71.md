---
title: 'HomeBrewで Warning: ... :macos => :yosemite is deprecated! ...と出た'
emoji: 😗
type: tech
topics:
- mac
- homebrew
- mas
published: true
---

## はじめに

Macのアプリ環境を整備するためにHomeBrewを使ってBrewfileを作って色々インストールした経緯がある。
ある日以降、HomeBrewのupdateコマンドを実行した時に何やら見慣れないWarningが出てくるようになった。

```text
Warning: Calling depends_on :macos => :yosemite is deprecated! There is no replacement.
Please report this issue to the argon/mas tap (not Homebrew/brew or Homebrew/homebrew-core):
  /opt/homebrew/Library/Taps/argon/homebrew-mas/mas.rb:8
```

しばらく前から表示されておりエラーではないので後回しにしていたが、視界に入るのが気になってきたため原因を調査してみた。

## 環境

* Mac Mini(M1) Mac OS 13.3.1 Ventura
* HomeBrew 4.0.13
* mas-cli ?????（後述するが再インストールしてしまったため確認できない）

## 用語

* yosemite→MacOS Xの名前らしい
* mas→App Store上のアプリをパッケージ管理できるやつ

つまりはMac App Storeをパッケージ管理できるmasモジュールが古いOSの何かを使っており、それがある時から非推薦となった。
結果、`brew update`コマンド実行時にmasを呼び出すためにWarningが出るようになった。
ということね。

## 結論

masモジュール再インストールすることで出なくなった。

```bash
brew uninstall mas
brew install mas
```
