---
title: WSL2のUbuntuでrootless-dockerを動かす
tags:
  - Windows
  - Linux
  - Ubuntu
  - Docker
  - WSL2
private: false
updated_at: '2024-06-23T00:35:07+09:00'
id: edc09637d5e613a4f4b6
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

ローカルでの開発環境でWindowsのDocker Desktop（WSL2を使用）を使用していたが、このDockerはrootで実行されている。
手間などを考えるとrootは便利ではあるが、セキュリティ的にrootは使わない方がより安全となる。
いろんな記事を見ていたらrootを使わないdockerを使う方法があったので実際に作ってみた。
しかし構築手順で詰まったところがあったので、それらを考慮した手順と解決方法について解説する。

## 前提条件

+ Windowsが動作している
+ WSL2が有効になっている
+ ストア版で最新版のUbuntuをインストール（昔のバージョンではsystemdが使えないため最新版をインストールすることを推奨）
+ ターミナルはお好みでどうぞ（自分はWindows Terminal）

## WSL2でrootless-dockerを動かすためのポイント

+ systemdが有効になっている必要がある。（systemctlコマンドを使用するため）
+ WSL2では一部カーネルモジュールが存在しないため、iptables関連は一旦スキップする
+ rootless dockerがpullする時にWSL2が自動的に作るresolv.confで名前解決ができないため自分で定義

## 設定手順

### WSLの設定変更

まずUbuntu環境の/etc/wsl.confを開き以下の内容を書き込む

```conf:/etc/wsl.conf
[boot]
systemd=true

[network]
generateResolvConf=false
```

設定を反映させるため、WSLを再起動（というより停止）する。

```bash
#１つのみ停止する場合
wsl.exe -t <対象のディストリビューション名>

# 全て停止する場合
wsl.exe -shutdown
```

※Windows、WSLのどちらでも実行可。Windows側の場合は「.exe」は省略可能。

対象のディストリビューション名を調べる方法は

```bash
wsl.exe -l
```

### DNSリゾルバーの設定を行う

```conf:/etc/resolv.conf
# Cloudflare DNS
nameserver 1.1.1.1
```

※WSL2環境において手動でnameserverを変更するのは非推薦との記載がどこかにあった気がする。
もしこれの詳細を知っている方がいれば教えてください。

設定後、もう一度WSLを再起動。（先ほどと同じ手順なので説明は省略）

### Dockerが入っている場合、無効または削除

再びUbuntu環境に入り、下記コマンドで現在Dockerが実行できる状態にあるか確認する。

```bash
docker info
```

もし入っていた場合は無効にする。

```bash
sudo systemctl disable --now docker.service docker.socket
```

### 必須パッケージのインストール

```bash
sudo apt install uidmap
```

### docker用のユーザー作成

rootless-dockerを動かす専用のユーザー「docker_user」を作成し、パスワードを設定

```bash
sudo useradd -m -d /home/docker_user -s /bin/bash docker_user
sudo passwd docker_user
```

以降の作業はこのユーザーで行う

```bash
su docker_user
```

### rootless-dockerをインストール

```bash
export SKIP_IPTABLES=1
curl -fsSL https://get.docker.com/rootless | sh
```

### rootless-dockerの環境変数を設定

このインストールの最後で環境変数を追加する指示があるため設定する。
ファイル内の最後に追記する

```bash:~/.bashrc
export PATH=/home/docker_user/bin:$PATH
export DOCKER_HOST=unix:///run/user/<docker_userのuid>/docker.sock
```

設定を反映させる

```bash
source ~/.bashrc
```

### docker.serviceの設定変更

`~/.config/systemd/user/docker.service`を編集して`ExecStart=/usr/bin/docker-rootless.sh`の引数である`--iptables=false`部分を削除。

```service:~/.config/systemd/user/docker.service
ExecStart=/usr/bin/dockerd-rootless.sh --iptables=false
```

↓

```service:~/.config/systemd/user/docker.service
ExecStart=/usr/bin/dockerd-rootless.sh
```

設定を反映させる

```bash
systemctl --user daemon-reload
systemctl --user restart docker
```

## 動作確認

```bash
docker run hello-world
docker run -p 8000:80 nginx
```

### ルートレスモードか確認

```bash
docker context use rootless
# 「Current context is now "rootless"」と出てればOK

docker info
# SecurityOptionsに「rootless」が含まれていればOK

systemctl --user status docker
# Active: Active(running) と書いてあればOK


ps aux | -grep dockerd
# Docker関連のプロセスがdocker_userによって実行されているか確認
# rootで実行されていたら停止忘れなどが疑われる
# 他ユーザー名が出てたらどこかの手順でユーザーを変え忘れている
```

### 他ユーザーからdockerが見えるか

```bash
docker info
# 起動状態とはなっていないはず
```

## 自動起動設定

起動に成功したら以降は自動で起動するようにする

```bash
systemctl --user enable docker

# 他ユーザーのログイン時でもdockerが起動するようにする
sudo loginctl enable-linger docker_user
```

## 通常のDockerと違う点について

### rootless-dockerの制約

#### 使用できるポートに関する制限

通常（rootfull）のdockerの場合はhttpサーバーのポート80が使えたが、root権限がないとこのポートは使えない。
例えば以下のコマンドでnginxをそれぞれ動かし、Windowsからページが表示できるか確認してみる。

```bash
# 実行できるがhttp:localhostは開けない（実行はできてしまうので沼にはまりそう）
docker run nginx

# 実行できない
docker run -p 80:80 nginx

# 実行でき、http:localhost:8000にアクセスできる
docker run -p 8000:80 nginx
```

このことから他と競合しないポートを常に指定する癖をつけた方が良いだろう。
念のため言っておくが、`docker run`コマンドの他、`docker-compose.yml` への記載も同様。

### WSL2のrootless-dockerならでは制約

#### Docker Desktopからの管理はできない

GUIに慣れた人にとっては辛いかもしれないがDocker Desktopからの操作はできない。
これはDocker DesktopからWSL2内のDockerを利用する場合にWSLがroot権限でDockerを実行するようになっているため。
もしユーザー権限のDockerを使用できるとしたら、Docker Desktopの画面にWSL2ディストリビューションを選ぶ他にユーザーを選択することができるはず。

話は変わるが、WSL2のDockerでWindowsのファイルシステムにボリュームマウントを行った場合にLinux側からのアクセスが非常に遅くなる。
Linuxファイルシステムにボリュームマウントした場合はこの問題は起こらない。
それもあって自分はDocker Desktopは使わずにWSL2の中でDocker CLIを使って開発するように移行している。

## トラブル一覧

### systemctlが使用できない

Ubuntu（アプリ）を最新版にする前に発生。
systemdが使えない頃にインストールしたため使えなかった。

### Dockerインストール時、iptablesが無いと言われる

このコマンド実行時に発生

```bash
curl -fsSL https://get.docker.com/rootless | sh
```

iptablesモジュールはWSL2のカーネルには存在するものの、Dockerが想定する本来の場所に存在せず別な場所に存在する。
そのため使わない設定でインストール後、もとに戻す。
なお本来の場所は`/lib/modules/<カーネルビルド名>/modules.builtin`にあるはず。
WSLのUbuntuではこの`/lib/modules`が存在しない。
カーネルをカスタムビルドすればこれを入れることもできるらしいが手順は知らない。

### rootless-dockerを起動後、Docker Hubからpullできない

`docker info`実行はできるのに、`docker run nginx`ができない。
出てきたエラーメッセージは10.0.2.3に接続できないというもの。
どうやらこのIPアドレスはVirtualBoxのネットワークのデフォルトゲートウェイアドレスらしい。
VirtualBoxは今回使ってないので仮想環境全般でDockerが動く際にそれを見に行ってるのかな？

まずWSL2でネットに接続する仕組みの話になるが、ホストOSに192.168.192.1というイーサネットアダプタが追加され、`/etc/wsl.conf`にそれを参照する設定が自動的に組まれている。

```conf:/etc/resolv.conf
# This file was automatically generated by WSL. To stop automatic generation of this file, add the following entry to /etc/wsl.conf:
# [network]
# generateResolvConf = false
nameserver 192.168.192.1
```

これによってホスト（Windows）を経由してインターネットへの接続を行っている。
ホスト（Windows）のネットワーク設定を見るとWSLのネットワークがあるはず。

通常Ubuntu側がネットを参照する際ホストを経由するわけだが、仮想環境のDockerがネットワークに繋ぐ設定は「仮想環境だったら10.0.2.3で名前解決する」みたいな設定になっているのかもしれない。

Dockerイメージが置いてあるURLを名前解決するために`/etc/resolv.conf`を見る
ホストOS（192.168.192.1）にDockerHubのURLについて尋ねるが、そんなURLは知らない
イメージのpullができないじゃん

↓`/etc/resolv.conf`の変更

Dockerイメージが置いてあるURLを名前解決するために`/etc/resolv.conf`を見る
Cloudflare DNS（1.1.1.1）にDockerHubのURLについて尋ねて応答があったのでアクセスしてpullする

となったのかな？

ただしこの設定はセキュリティ的に微妙っぽいという話も聞いた。
仮想環境から直接ネットに繋がるのが微妙ということだろうか。

この問題だけでかなりの時間を浪費した。
しかもなぜ解決できたのかの理由がまだ完全には理解できていない。

## まとめ

まだ使い込んだわけではないので実運用に耐えるものなのかは不明だが、とりあえず動作させることはできた。

rootじゃなくなったからといってセキュリティの問題が全て解決するわけではないので、どこの誰が作ったかもわからないようなイメージを実行したりしないよう注意は必要。
あくまでもセキュリティ上の懸念の1つが解消しただけ。

もし、これを使っていく上でまたトラブルがあったら追記します。

## 参考にした記事

<https://k-hyoda.hatenablog.com/entry/2020/09/20/235346>
<https://e-penguiner.com/rootless-docker-for-nonroot/>
<https://qiita.com/shigeokamoto/items/f09d6fead8d99bbf4e3b>
<https://github.com/microsoft/WSL/issues/7466>
