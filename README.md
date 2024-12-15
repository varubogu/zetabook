# zetabook

[Zeta](https://github.com/TyomoGit/zeta) を使用してQiitaとZennの記事管理を行うためのDev Containerです。

使用したい方は下記のリポジトリをfork、あるいはdevcontainerだけをコピーして環境を構築してください。（このリポジトリからforkしないでください）
<[https://github.com/varubogu-organization/zeta_contaienr](https://github.com/varubogu-organization/zeta_container)>

必要なものは下記になります。

- Dev Container対応のエディター（VS Code、Cursor、JetBrainsIDEなど）
- Docker（devcontainerを動かすため）
- WSL2（Windows環境のみ、Dockerのため）

## 使い方

devcontainerをビルドする時にNodejs、Cargo、Zetaの最新版のインストールを自動的に行います。
なので、VS Codeなどで「.devcontainerディレクトリ」のあるディレクトリを開き、そこからDev Containerとして開けば準備がほぼ整います。
あとはお好みで拡張機能をインストールしてください。筆者が使っている拡張機能は".devcontainer/devcontainer.json"に設定としていれてあります。
[zetaの紹介記事](https://code.visualstudio.com/docs/devcontainers/containers)にある通り、Run On Saveという拡張機能も推薦拡張機能としており、設定は".vscode/settings.json"に入っているため拡張機能を入れるだけで使えます。

## 注意点

[Zetaの紹介記事](https://zenn.dev/pullriku/articles/article-batch-management)にもある通り、privateなリポジトリだと画像の取得ができません。
publicなリポジトリにするか、別な場所に画像をアップロードしてそのURLを参照する形にしてください。

## 採用技術およびライセンス

- [Zeta](https://github.com/TyomoGit/zeta)
  - ライセンス:[MIT License](https://github.com/TyomoGit/zeta?tab=MIT-1-ov-file)
- [Docker](https://www.docker.com/ja-jp/)
- [VS Code](https://github.com/microsoft/vscode)
  - ライセンス:[MIT License](https://github.com/microsoft/vscode?tab=MIT-1-ov-file)
- [Dev Container](https://code.visualstudio.com/docs/devcontainers/containers)
  - ライセンス:[MIT License](https://github.com/microsoft/vscode?tab=MIT-1-ov-file)
- [vscode-runonsave](https://github.com/emeraldwalk/vscode-runonsave)
  - ライセンス[Apache-2.0 License](https://github.com/emeraldwalk/vscode-runonsave?tab=Apache-2.0-1-ov-file)
