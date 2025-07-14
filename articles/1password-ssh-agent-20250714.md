---
title: 【2025年7月最新版】1PasswordのSSHエージェントを設定する
emoji: 🔑
type: tech
topics:
- 1Password
- SSH
- SSH Agent
- 生体認証
published: true
---

## はじめに

1PasswordによるSSHエージェント機能が実装されてからしばらく経ち、色々なアップデートが加わり以前よりも使いやすくなった。
それに伴い色々とやり方が変わったのだが最新のやり方に関する記事が見当たらなかったので、2025年7月時点での最新情報をまとめておきます。

## 対象読者

- SSHを使っている方
- 1Password SSHエージェントの最新版の設定方法を知らない方

## 今回は説明しないこと

- SSHと鍵に関する基本的なこと（使い方、鍵ファイルの種類、コマンドなど）

## 1PasswordのSSHエージェントとは

1PasswordのSSHエージェントは、SSHキーを安全に管理しつつSSH接続時にパスフレーズの代わりに生体認証を行うことで自動的にキーを提供する機能です。
これにより、SSHキーのパスフレーズを毎回入力する必要がなくなってミスが減るだけでなくセキュリティも向上します。

## 最新版：1PasswordのSSHエージェント設定方法

1. Windowsの場合、Open SSH Agentサービスを停止する
2. WSL環境の場合、ホストOSのSSHを使う設定をする
3. 1Passwordを開き、設定 > 開発者メニューのSSH関連の項目をONにする
4. SSH configの編集
5. 各SSHキーのアイテムに対して「URL」項目を追加する
6. 試しにSSH接続してみる

1つずつ解説します。

### 1. Windowsの場合、Open SSH Agentサービスを停止する

WindowsまたはWSL環境でOpenSSHを使っており、なおかつ1PasswordのSSHエージェントを使用する場合、Open SSH Agentサービスを停止する必要があります。
これは、1PasswordのSSHエージェントが独自にSSHエージェント機能を提供するため、Open SSH Agentとの競合を避けるためです。
サービスを停止するには、以下の手順を実行します。

1. Windowsの「サービス」を開く
2. 「OpenSSH Authentication Agent」を探す
3. 右クリックして「サービスの停止」を選択
4. プロパティを確認し、「スタートアップの種類」を「無効」に設定

![WindowsのSSHエージェントを停止](/images/1password-ssh-agent-20250714/3.png)

これによって常に1PasswordのSSHエージェントが優先されます。

### 2. WSL環境の場合、ホストOSのSSHを使う設定をする

WSL環境でのSSHはホストOSのSSHとは違い、WSL環境のSSHとなっています。
当然ながら設定も別物になっており、このままだとホストOSの1Passwordの生体認証を経由した処理ができません。
しかし、実はWSL環境からでもホストOSのexeを呼び出すことができます。
試しにWSL環境でこのコマンドをやってみてください。

`which ssh`（WSL環境のSSH）

```bash
which ssh
```

結果

```txt
/usr/bin/ssh
```

`which ssh.exe`（Windows環境のSSH）

```bash
which ssh.exe
```

結果

```txt
/mnt/c/windows/System32/OpenSSH//ssh.exe
```

つまり`ssh.exe`コマンドを使えばホストマシンの生体認証環境がそのまま使えます。
しかし毎回`ssh.exe`と打つのも面倒です。
ホストOSのSSHが`ssh`のみで使われるように`~/.bashrc`などにエイリアスとして記載しましょう。

一時的に試す場合

```bash
alias ssh=ssh.exe
alias ssh-add=ssh-add.exe
```

今後ずっと利用する場合は、上記を手動で編集に行くか以下のコマンドを実行する

```bash
echo "alias ssh=ssh.exe" >> ~/.bashrc
echo "alias ssh-add=ssh-add.exe" >> ~/.bashrc
```

エイリアス登録後は元の`ssh`などのコマンドは使えなくなりますが、どうしてもWSL環境のを直接使いたい場合はコマンド実体を直接呼び出せば本来のsshなどが使えます。

```bash
/usr/bin/ssh -T git@github.com
```

```bash
/user/bin/ssh-add -l
```

### 3. 1Passwordを開き、設定 > 開発者メニューのSSH関連の項目をONにする

1Passwordを開き、左側のメニューから「設定」>「開発者」を選択します。

「SSHエージェントを使用」と「ブックマークされたホストを含むSSH設定ファイルを生成する」を見つけてそれぞれ有効にします。

![1PasswordのSSHエージェント設定](/images/1password-ssh-agent-20250714/1.png)

### 4. SSH configの編集

2で項目をONにした場合、`~/.ssh/1Password/config`ができているはずです。
このファイルは、1Passwordが自動的に作り出すSSHの設定になります。
その後 `~/.ssh/config` に上記ファイルを読み込むように `Include ~/.ssh/1Password/config` という記載を追加します。

```~/.ssh/ssh.config
Include ~/.ssh/1Password/config
```

### 5. 各SSHキーのアイテムに対して「URL」項目を追加する

「ssh://ユーザー名@接続先ホスト名」を設定します。
カテゴリの中に作ったり、項目名の設定の必要はありません。
URLが「ssh」プロトコルであれば認識してくれます。

GitHubの場合

```url
ssh://varubogu@github.com
```


これをすることにより、`~/.ssh/1Password` ディレクトリにそれぞれの公開鍵ファイルが配置され、`~/.ssh/1Password/config`ファイルに設定が書き込まれます。

### 6. 試しにSSH接続してみる

試しにGitHubにSSH接続してみましょう。（もちろん他に接続先があるならそちらでも大丈夫です）

```bash
ssh -T git@github.com
```

ここまでの設定がうまくいっていれば1Passwordの認証画面が出てきて、生体認証あるいはパスワード入力によるロック解除をしたら認証が成功するはずです。

失敗した場合は以下を確認してください。

- これまでの手順の見直し
- `~/.ssh/config`に`~/.ssh/1Password/config`を打ち消すような定義がないか
- 1Passwordは起動しているか
- Windowsの場合、OpenSSH Agentサービスを終了しているか
- WSLの場合、`ssh.exe`でも失敗するか

![1PasswordのSSHエージェント生体認証](/images/1password-ssh-agent-20250714/2.png)

```message
Hi varubogu! You've successfully authenticated, but GitHub does not provide shell access.
```

## 以前から使っていた方向けの解説

### 「Too many Authentication failures」（認証失敗が多すぎる）というエラーがなくなった

以前は「Too many Authentication failures」（認証失敗が多すぎる）というエラーが出ることがありました。
これは当時SSHエージェントがどのサイトに対してどのキーを使うかというのがわからないため、候補となるキー順番に1つずつ試していたためです。
SSHキーが少ないうちは良いですが、様々なサーバーを管理していく過程でSSHキーが増えてしまい、接続時に多くのキーを試す必要が出てきました。
大体のサーバーでは6回失敗すると接続が拒否されるため、それ以上の試行ができませんでした。
しかも公開鍵とはいえ片っ端から送るのはどうなのという気もしてしまいます。

以前はこの問題に対して、公開鍵の設置＋SSHの設定ファイル（`~/.ssh/config`）に以下のような設定を追加することで解決していました。

```txt
Host example.com
  HostName example.com
  User varubogu
  IdentityFile ~/.ssh/id_rsa_example.pub # 公開鍵のパス
  ForwardAgent yes
```

まず前提として、1つの接続先（SSHキー）ごとに秘密鍵と公開鍵が対になっています。
そのうちホストに対する公開鍵を指定しておくことで、それを目印として1Password内を検索し、1Passwordの公開鍵情報から該当する鍵を抽出することで今までは検出していました。
しかし欠点として、接続先が増えるたびに手動で`~/.ssh/config`を編集する必要がありました。（1Passwordを使わない場合はそれが当たり前でしたが）
1PasswordのSSHエージェントが改良されてsshのconfigが自動的に作られるようになったされたことで、この方法は不要になりました。

### WSL環境でも対して手間なく使えるようになった

1PasswordのSSHエージェント機能が出たばかりの頃は、WSL環境での1Password利用は「npiperelay.exe」というものを使ってSSHのリクエストをWSLからWindows側に転送していました。
それが今は「ssh」と「ssh-add」を「.exe」をつけたエイリアスにすることでホスト側のexeを直接呼び出せるようになりました。

## さいごに

これで今後は1PasswordにSSH鍵の管理を委ねることができ、ローカルに秘密鍵を置かなくてもよくなりました。
さらに手間が従来の手順よりも大幅に簡略化されていて、作業効率があがることでしょう。

もし今後1Passwordに移行したいと考えている場合、ローカルの鍵情報を1Passwordに取り込む機能や、`~/.ssh`の中に秘密鍵が残っているかスキャンする機能もあります。

余談にはなりますが、SSHの鍵を作る時のアルゴリズムは現在「ED25519」がオススメです。
RSA（4096bit）よりも暗号強度がより高くて高速な新しい方の規格です。
ただし古いSSHサーバーでは対応していないかもしれません。
そういった場合には従来通りのRSA（4096bit）を使いましょう。
