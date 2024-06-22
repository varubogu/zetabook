---
title: Mac（というよりEmacs）っぽい操作感とWindowsショートカットを両立してみた
tags:
  - Emacs
  - Windows
  - AutoHotkey
  - Google日本語入力
  - Keyboard
private: false
updated_at: '2024-06-22T23:56:19+09:00'
id: 6d462a3b9023f3cdbf4e
organization_url_name: null
slide: false
ignorePublish: false
---

# Mac（というよりEmacs）っぽい操作感とWindowsショートカットを両立してみた

## はじめに

突然だが皆はキーボードのホームポジションは適切に使えているだろうか。
自分はというと専門学生のころにタイピング練習用のゲーム（名前は確か「特打」）をやって以来、ホームポジションを崩さずにブラインドタッチを徹底できるようになった。

しかし、実際にコーディングをしていたりすると漠然と以下の２つの悩みがあった。

- 矢印キー、Home、Endなどホームポジションから離れすぎるキーがある
- 左手小指で押すキーの中でCapsLockだけ使い道がない

### MacBook（JIS配列）との出会い

そんな中、とある友人がMacBookを格安で譲ってくれることとなり、Macも触っておくかーという軽い気持ちで譲ってもらうことにした。

MacだとCtrl+●と打つことでカーソルを動かすことができ、しかもWindowsでいうCapsLockの位置（左手小指）にCtrlキーが割り当てられている。
これなら、カーソル操作などをするときにホームポジションを崩す必要がない。

後々調べた結果、その操作は「Emacsキーバインド」と呼ばれるもので、ホームポジションから手を離すことなく操作することを念頭に作られたエディタがVimとEmacsだった。

また、Macだと現在の入力状態（アルファベットか、ローマ字か）を意識せず、日本語キーと英語キーがあるので切り替えがしやすい。

この経験からMacやEmacsキーバインドの操作感が好きになり、Windowsでも実行したいと思ったのである。
しかし、単純にC-f（Control押しながらfキー）やM-f（Altキーを押しながらfキー）で実装するとアプリごとのショートカットが使えなくなってしまう。
Windowsショートカットも使いたいものも中にはある。
そこで普段使っていない右Ctrlと右AltをEmacsのキーとして割り当てることにした。

## 設定

ここで設定すべき項目は大きく３つある

- CapsLockキーの動作を変更する
- 変換・無変換キーの動作を変更する
- Emacsキーバインドを設定する

### CapsLockキーの動作を変更する

まずはCapsLockキーの動作を変更する。
Mac（JIS配列）ではControlキーの位置となっている。
そのためCapsLockキーに右Ctrlキーを割り当てるのが良いだろう。

CapsLockキーの動作変更はAutoHotKeyでうまくいかないケースがあるらしい。
確実に置き換えるためにレジストリを変更してキーの割当を変える。
しかしレジストリを直接変更するのはなんとなく嫌なので、KeySwapというソフトで行う
![CapsLockキーの動作変更（KeySwap）](emacs-like-rightkey-autohotkey-2.png)

### Google日本語入力

次にMac（JIS配列）と同じように
変換キー→かな
無変換キー→英数
に変更する。

ここでわざわざGoogle日本語入力で設定いることについては、単純にAutoHotKeyで変更してしまうと無変換キー、変換キーが使用できなくなるのを防ぐため、入力していない時のみ切り替え可能にする。

設定>一般>キー設定>キー設定の選択>編集

モード「入力文字なし」の
「Henkan」を「ひらがな」に変更
「Muhenkan」を「IMEを無効化」に変更

![Google日本語入力の設定](emacs-like-rightkey-autohotkey-1.png)

### AutoHotKeyスクリプト

今回AutoHotKeyのスクリプトは自作した。
先駆者は結構いるのだが、2022年にAutoHotKeyのバージョンがv2に上がっており、せっかくだし自分で１から作成し直すことにした。
修正したものについてはGitHubに公開している。

https://github.com/varubogu/emacs-like-rightkey.ahk


### Emacsキーバインド

Emacsのキーバインドは以下の通りに設定。

https://github.com/varubogu/emacs-like-rightkey.ahk/blob/main/README.ja.md


## 余談①他キーを割り当てる方法もある

ControlではなくF13以降のキー（大抵のキーボードには存在しない）やPowerボタン（一部キーボードのみ存在）を右Ctrlや右Altの代わりに割り当てる方法もある。

## 余談②なんのキーボードを使ってる？

Realforce R2の静音、黒、全キー30gのやつを使ってる。
静電容量無接点方式信者。
外出用にHHKBの方も欲しい。
US配列も触ってみたさはある。

## さいごに

これでMacっぽい操作感とWindowsショートカットを両立することができた。
WindowsとMacで度々間違うことも減るだろう。（コピペとかは未だによく間違うけど・・・）
