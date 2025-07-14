---
title: 1PasswordのSSHエージェントが色々変わっていた
tags:
- 1Password
- SSHエージェント
private: false
updated_at: ''
id: ''
organization_url_name: null
slide: false
ignorePublish: true
---

## はじめに

1PasswordによるSSHエージェント機能が実装されてからしばらく経ちましたが、最新のやり方に関する記事が見当たらなかったので、2025年7月時点での最新情報をまとめておきます。

## 1PasswordのSSHエージェントとは

1PasswordのSSHエージェントは、SSHキーを安全に管理し、SSH接続時に自動的にキーを提供する機能です。これにより、SSHキーのパスフレーズを毎回入力する必要がなくなり、セキュリティと利便性が向上します。

## 1PasswordのSSHエージェントの設定方法

1. 1Passwordを開き、設定メニューに移動します。
2. 「SSHエージェント」オプションを見つけて有効にします。
3. SSHキーを1Passwordに追加し、必要に応じてパスフレーズを設定します。
4. ターミナルを再起動し、SSH接続を試みます。1Passwordが自動的にキーを提供するはずです。

## 鍵を作って接続する

1PasswordのSSHエージェントを使うためには、まずSSHキーを作成する必要があります。
コマンドがわかる方であればコマンドで良いですし、GUIのほうからも作成できます。


## 以前使っていた人向けの解説

以前は「Too many Authentication failures」（認証失敗が多すぎる）というエラーが出ることがありました。
これは当時SSHエージェントがどのサイトに対してどのキーを使うかわからないため候補となるキーを全て試していたためです。
SSHキーが少ないうちは良いですが、様々なサーバーを管理していく過程でSSHキーが増えてしまい、接続時に多くのキーを試す必要が出てきました。
しかし大体のサーバーでは6回失敗すると接続が拒否されるため、それ以上の試行ができませんでした。

以前はこの問題に対して、公開鍵の設置＋SSHの設定ファイル（`~/.ssh/config`）に以下のような設定を追加することで解決していました。

```plaintext
Host example.com
  HostName example.com
  User varubogu
  IdentityFile ~/.ssh/id_rsa_example.pub # 公開鍵のパス
  ForwardAgent yes
```

まず前提として、1つの接続先ごとに秘密鍵と公開鍵が対になっている。
そのうち公開鍵を目印として1Password内を検索し、1Passwordの公開鍵情報から該当する鍵を抽出することで今までは検出していました。
しかし、1PasswordのSSHエージェントが改良されてsshのconfigが自動的に作られるようになったされたことで、この方法は不要になりました。

## さいごに