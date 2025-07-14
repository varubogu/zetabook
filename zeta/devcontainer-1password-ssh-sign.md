---
title: 'Dev containerでの1Password SSH Agentを使ったコミット署名時のエラーを対処する'
emoji: 💻
type: tech
topics: ['devcontainer', 'sign', '1Password', 'ssh', 'gpg']
qiita_id: '5b3014871b0a71fe8cc4'
published: true
---

## はじめに

1Passwordのssh agentは生体認証でコミット署名できる点がいいよね。
しかし、1Password推奨の方法で`~/.gitconfig`を設定すると、そのマシン上で操作する分には困らないがVS CodeのDev container内でコミット署名をする際には、以下のようなエラーが発生する。

```message
Git: fatal: cannot exec 'C:/Users/ユーザー名/AppData/Local/1Password/app/8/op-ssh-sign.exe': No such file or directory
```

そこで今回は、その仕組みについての解説と、具体的な改善案がいくつか出てきたので、それをまとめてみる。

## 対象読者

- VS CodeのDev containerを使っている人
- 1PasswordのSSH Agent
  - 関連する話題として、GPG Agentも同様の問題が発生する
- それらが動作する各種OS

## 根本原因

まず結論を先に言うと、Dev container内でコミットする際にコンテナー内のgitが参照する設定ファイル`~/.gitconfig`がホスト側からコピーして持ってきているというのが根本的な原因だ。

### .gitconfigのコピーの挙動について

まず大前提として、Dev containerを作る際、通常はホストの`~/.gitconfig`をそのままコピーしている。
これはVS Codeのオプション
設定 > 拡張機能 > 開発コンテナー > Dev Containers: Copy Git Config
の初期値がtrueになっているためこのような挙動をする。
このオプションは以下の順番でコピー元を自動的に探し、最初に見つかった方をコンテナーにコピーして使用する。

1. `~/.gitconfig` ※デフォルトのパス
2. `~/.config/git/config` ※XDG BASE Directoryに則ったパス

2のパターンが見つかった場合、Dev containerでもそのディレクトリ構造が保持されるため、Dev container内でも`~/.config/git/config`になる。

これらのパターンが存在しない場合にマシン単位の`/etc/git/config`などがコピー対象になるのかは不明。

コピーする場合は内容が丸ごとコピーされるため、1Passwordのssh agentへのパス設定も込みでホストOSのものがコピーされてしまう。
Dev containerのGitがこのconfigを読み込んでSSHを実行したりする場合にそんなパスは存在しないためエラーが発生する。

以下はWindowsで1Password推奨の`~/.gitconfig`定義のスニペットだ。

```~/.gitconfig
[user]
  signingkey = ssh-ed25519 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

[gpg]
  format = ssh

[gpg "ssh"]
  program = "C:/Users/ユーザー名/AppData/Local/1Password/app/8/op-ssh-sign.exe"

[commit]
  gpgsign = true
```

この定義をホストマシン（Windows）で設定した状態でDev containerを作り、コミットしようとすると以下のエラーが発生する。

```message
Git: fatal: cannot exec 'C:/Users/ユーザー名/AppData/Local/1Password/app/8/op-ssh-sign.exe': No such file or directory
```

エラーの原因になっているのは下記のgpg.ssh.programの行だ。

```.gitconfig
[gpg "ssh"]
  program = "C:/Users/ユーザー名/AppData/Local/1Password/app/8/op-ssh-sign.exe"
```

コンテナー内で`~/.gitconfig`を開き、上記行をコメントアウトすればエラーは解消され、ホストマシンを経由して生体認証にアクセスされ、コミット署名もできるようになる。
ホストと同期はしていないため、コンテナーごとにその作業が必要になる。

この対処方法はWindows以外のOSでも同じ。

このエラーについては1PasswordのSSH Agentに限った話ではなく、GPG Agentを使っている場合も発生する。
エラーの根本原因は`~/.gitconfig`に存在しないパスが書かれているということであるためだ。
以下のような記述があれば同様の問題が発生するはずだ。

```~/.gitconfig
[gpg]
  program = "C:/Program Files (x86)/GnuPG/bin/gpg.exe"
```

### ホストの設定を利用する際の仕組み

Dev container内からコミット署名やSSH接続を行う際、ホスト側の構成を利用する設定値は自動的に書き込まれるようになっている。
これは先ほどの`~/.gitconfig`のコピーとはまた別のオプションとして存在する。
設定 > 拡張機能 > 開発コンテナー > Dev Containers: Git Credential Helper Config Location
初期値は`global`で、`~/.gitconfig`に書き込まれるようになっている。

```gitconfig .gitconfig
[credential]
    helper = "!f() ※長いので省略
```

また、Gitがconfigを読み込む際は必ず

1. `~/.config/git/config` ※XDG BASE Directoryに則ったパス
2. `~/.gitconfig` ※デフォルトのパス

の順に読み込みが行われるようになっている。
それぞれの設定がマージされ、キーが被る項目があった場合、優先されるのは`~/.gitconfig`だ。

## 解決策について

### 解決策1: ホスト側のgitconfigを分割し、SSHエージェントの設定を別ファイルにする

どのような人にオススメか: 極力シンプルな設定にしておきたい人

gitconfigのコピーは行われるが、その中で`include`によって別ファイルから設定値を読み取ってる場合、そのファイルのコピーまでは行われない。
また、存在しないファイルを`include`してもエラーにはならないため、この仕組みを使ってマシンに依存する設定を分割することで、Dev container内にコピーされても問題ない`.gitconfig`が使用できる。

たとえば以下のようにユーザー単位の設定と、マシンごとの固有設定をそれぞれ記述する

自分（ユーザー）用の設定

```.gitconfig
[user]
    name = hoge
    email = hoge@example
    signingkey = ssh-ed25519:hogehoge
[commit]
    gpgsign = true
[gpg]
    format = ssh
[init]
    defaultBranch = main
[include]
    path = .gitconfig.local
```

マシンごとに固有の設定

```.gitconfig.local
[core]
    sshCommand = "C:/windows/System32/OpenSSH/ssh.exe"

[gpg "ssh"]
    program = "C:/Users/ユーザー名/AppData/Local/1Password/app/8/op-ssh-sign.exe"
```

ここで挙げた構成はあくまで一例であり、もっと細かい調整をすることもできる。
たとえば、nameとemailだけ使いまわしたいが、signingkeyだけマシンごとに変えたい場合、signingkeyの項目をさらに別な`include`で読み込むこともできる。
ただし、その場合でもコピーされるものは`.gitconfig`のみであるため、`include`対象のファイルはコピーされない。
そのような単純でない構成を使っている場合は、次に挙げる解決策2を検討するほうが良いかもしれない。

### 解決策2: gitconfigのコピーを無効化する

どのような人にオススメか: 複雑な設定をしている人、あるいはdotfilesを使っている人

VS Codeのオプションをそもそもオフにすることで、ホストのgitconfigをDev containerにコピーしないようにすることもできる。
その場合、Dev container内でのgitconfigの設定は、Dev container内で直接設定する必要がある。
コンテナーを作るたび、ビルドするたびに毎回設定をする必要があるが、スクリプトで自動化することもできる。
たとえばDotfilesリポジトリがある場合、VS Codeの設定でdotfilesの自動クローン機能とインストール機能があるためそれを使ってセットアップすることができる。

- 設定 > 拡張機能 > 開発コンテナー > Dotfiles: Repository

dotfilesリポジトリのURLを入力する。
クローンは後述のパスに対して行われる。

```bash
https://github.com/hoge/dotfiles.git
```

- 設定 > 拡張機能 > 開発コンテナー > Dotfiles: Target Path

dotfilesリポジトリのクローン先のパスを入力する。

```bash
~/dotfiles
```

- 設定 > 拡張機能 > 開発コンテナー > Dotfiles: Install Command

dotfilesリポジトリをクローンした後に実行するコマンドを入力する。

```bash
~/dotfiles/install.sh
```

これらを駆使することで、Dev container内での設定を自動化することができる。

もしInstallCommandで`~/.gitconfig`をホームに展開するスクリプトを書いている場合、追記モードで実行すること。
そうでないと自動でVS Code（Dev container?）が書き込むホストへの転送設定まで上書きされてしまい、ホスト側への転送が効かなくなる。
これはXDG Base Directoryに則ったパス`~/.config/git/config`に展開する場合は発生せず、両立できる。

筆者はdotfilesを使っているため、この方法を使っている。

## まとめ

ここまで挙げた解決策のいずれかを使うことでコンテナーごとに毎回Gitのエラーを対処する必要がなくなるはずだ。

ここ2年くらいずっとDev container内の`.gitconfig`を直接書き換える手法で作業していたが、この方法を使えばもっと楽に設定できることがわかった。
たまにはVS Codeの設定を流し読みしてみるのもいいものだね。
