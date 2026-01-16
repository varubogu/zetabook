---
title: DockerでRust製Discord BotとPostgreSQLを使った開発・運用のための環境構築
emoji: rust
type: tech
topics:
- poise
- Rust
- DevContainer
- Docker
- GitHubActions
published: false
---

## はじめに

今までdiscord.pyで作ったBotをDockerで運用してきたが、Rust製のpoiseを使うとリソース効率が大きく改善する。
しかし今までビルドが必要なタイプのものをDockerで利用したことがなく、
かといってRustのビルドを本番サーバーでやらせると負荷が大きく、
他のアプリを動かしてたりする本番サーバーで負荷をかけるわけにもいかないので色々調べてみた。

ちなみに本番サーバー稼働費以外はいずれも料金がかからない構成としています。
※GitHub関連の場合、無料で使うには公開リポジトリである必要があります。

## 対象読者

- 開発から運用までDockerを活用したい方
- 利便性を損なわずにコンテナー開発環境を用意したい方
- Rustのようなビルドが必要なタイプのアプリをDockerで動かしたい方

※今回は環境構築に関する記事なので、poiseやrustコードの出番はあまりないです

## 取り扱う技術一覧

### 言語、ライブラリ、フレームワーク、DB

- Rust
- poise（serenity）
- PostgreSQL

### 開発環境

- Windows, Mac, Linux系いずれも可
- Docker（compose）
- Dev Container
- Docker Desktop※
※OrbStackやpodmanなどのDocker互換のコンテナエンジンなら大丈夫だと思います

### 本番サーバー

- Ubuntu24.04 ※Dockerが動くならサーバーOS問わず
- Docker（Rootlessモード）

### ビルドやデプロイ

- GitHub Container Registory
- GitHub Actions

## 開発環境（Dev Container）セットアップ

まずはdevcontainer.jsonとその周辺の設定ファイルから。

ざっくり言うとこんな構成です。

```txt
devcontainer.json
  └ docker-compose.yml
　　  └ appコンテナ
　　  └ dbコンテナ
```

.devcontainer/devcontainer.json

```json
{
    "name": "Rust and PostgreSQL",
    "dockerComposeFile": "docker-compose.yml",
    // docker-compose.ymlでDev Contaienrを作る場合はサービス名を指定する
    "service": "app",
    "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",
    "features": {
        // 開発環境の中で本番同様に試験をするためにDocker in Dockerを利用する
        "ghcr.io/devcontainers/features/docker-in-docker": "latest",
        // Node製ツールを使う場合は入れる。使わないなら無くて良い
        "ghcr.io/devcontainers/features/node": "latest"
    },
    // Dev Containerが作られた時に実行されるコマンド。シェルファイル化しておき汎用性を持たせる
    "postCreateCommand": "./.devcontainer/post_created.sh",
}
```

.devcontainer/docker-compose.yml

```yml
volumes:
  postgres-data:

services:
  app: # ★エディタで開かれるコンテナ
    build:
      context: .
      dockerfile: Dockerfile # RustのDevContainerだがimageを指定するやり方でも良い
    environment:
      RUST_LOG: debug # Rustのログをデバッグレベルで出したいために利用

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity

    networks: # dbコンテナと通信するために同じネットワークを指定
      - app-network

    # Use "forwardPorts" in **devcontainer.json** to forward an app port locally.
    # (Adding the "ports" property to this file will not forward from a Codespace.)

  db:
    build: # 検証や本番で使うDBコンテナと同一になるように一つ上のフォルダでDockerfile.dbをビルド
      context: ../
      dockerfile: Dockerfile.db # データ初期化スクリプトを含んだコンテナ
    restart: unless-stopped
    volumes: # データ永続化、Postgres18以降は/var/lib/postgresql/dataではなく１つ上の階層で永続化する必要がある
      - postgres-data:/var/lib/postgresql
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - "127.0.0.1:5432:5432" # ホストマシンからアクセスできるよう127.0.0.1:5432をコンテナの5432にバインド
    networks: # appコンテナと通信するために同じネットワークを指定
      - app-network

networks:
  app-network:
    driver: bridge
```

DevContainer.jsonからdocker-compose.ymlを参照することで、開発環境起動時に自動的にDBも立ち上がります。
起動は遅めですが手間のかからなさを重視しています。

ポート指定を「5432:5432」ではなく「127.0.0.1:5432:5432」とすることでホストマシンのポートをコンテナー内のポートにマッピングします。
これによりホストマシン上で動くDBクライアントでアクセスできるようになります。
「5432:5432」だけだとコンテナー内のpostgresのデフォルト設定で「同一ネットワーク以外の通信は受けない」ようになってるらしいです。
ホストマシンとDockerのネットワークは異なるため、これがホストマシンからはアクセスできません。
Docker Composeのコンテナー同士なら同一ネットワーク内なのでappやdbといったサービス名をホストに指定すればアクセスできます。

Dockerfile.db

```docker
# ================================
# PostgreSQL データベースコンテナ
# ================================
FROM postgres:18

# 初期化スクリプトをコピー（実行権限付き）
# docker-entrypoint-initdb.d 内の .sh ファイルは自動的に実行される
COPY --chmod=755 ./db/sh/init.sh /docker-entrypoint-initdb.d/init.sh
COPY --chmod=755 ./db/sql/init.sql /sql/init.sql
```

postgresコンテナーについてですが本番用と合わせるために1つ上の階層のDockerfile.dbをビルドしています。
そしてその中では初期化用のSQLと、それを呼び出すためのbashスクリプトを配置します。
sqlは初期化用のディレクトリとは別にしないとシェルスクリプトとSQLが両方実行されてしまうので、
/sqlに配置してシェルからはそこを呼び出すようにしています。

sqlファイルだけだと環境変数の参照が難しかったため、シェルスクリプトも併用しています。
これもいかにセットアップを楽できるかに重点を置いてます。

## 検証用、本番用のdocker-compose.yml

```yml
networks:
  app_network:
    driver: bridge

services:
  app:
    # 本番用：ビルド済みイメージをGHCRからpull
    # イメージ名は自動的に ghcr.io/<OWNER>/<REPO>:latest になります
    image: ghcr.io/varubogu/gbf_discord_bot_rs:latest

    # # 検証用：ビルドする
    # build:
    #   context: .
    #   dockerfile: Dockerfile.app
    restart: always
    env_file:
      - .env.app
    environment:
      DB_HOST: db
      TZ: UTC  # タイムゾーンを明示的にUTCに設定
    volumes:
      # Googleスプレッドシート操作のためのGoogle Service Account Keyファイルを読み取り専用でマウント
      - ./.local:/app/.local:ro
    networks:
      - app_network
    depends_on:
      db:
        condition: service_healthy  # DBのヘルスチェック完了を待つ
    logging:
      options:
        max-size: "10m"
        max-file: "3"

    # Use "forwardPorts" in **devcontainer.json** to forward an app port locally.
    # (Adding the "ports" property to this file will not forward from a Codespace.)

  db:
    # DBイメージもGHCRからpull ※説明はappの方を参照
    image: ghcr.io/varubogu/gbf_discord_bot_rs-db:latest
    # # 検証用：ビルドする
    # build:
    #   context: .
    #   dockerfile: Dockerfile.db
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql
    env_file:
      - .env.db  # .envから環境変数を読み込み
    networks:
      - app_network
    healthcheck: # コンテナが落ちてないかチェックする処理。DB定義と合わせる必要あり
      test: [ "CMD-SHELL", "pg_isready -U postgres -d postgres" ]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      options:
        max-size: "10m"
        max-file: "3"
volumes:
  pgdata:
```

本番用と検証用でyamlを分けている理由ですが、
検証はローカルでコンテナビルドして通ることを確認、
本番はGitHub ActionsでビルドしたものをGitHub Container Registoryで公開し、
それをpullする構成としているためです。
本番サーバーでRustのビルドをすると負荷がすごいことになってしまうためビルドと運用はサーバーを分けるべきです。

app→dbに対してdepends_onをすることでdbコンテナーが起動するまでappコンテナーは待機します。

1つ注意点として、この例では`.env`ファイルを利用していますが、平文でトークンをホストマシンに配置するということになってしまいます。
ホストマシン上に侵入されたら盗み見られるおそれもあるため、今後はもう一段階上のセキュリティにしたいところです。
具体的にはDocker Secret (Swarm)などを使う形としたい。

## GitHub Actions関連

```yml
name: Build and Push Docker Image

on:
  push:
    tags:
      - 'v*.*.*'  # v1.0.0 のようなバージョンタグのみ
  workflow_dispatch:  # 手動トリガーも可能にする

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    strategy:
      matrix:
        include:
          - arch: amd64 # AMDビルド用
            runner: ubuntu-latest
            platform: linux/amd64
          - arch: arm64 # ARMビルド用（本番環境はARMのサーバーなのでARMも追加した）
            runner: ubuntu-24.04-arm
            platform: linux/arm64

    runs-on: ${{ matrix.runner }}
    permissions:
      contents: read
      packages: write  # GHCRへのpushに必要

    steps: # この処理がARM,AMD環境でそれぞれ実行されます。
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest-${{ matrix.arch }}
            type=semver,pattern={{version}}-${{ matrix.arch }}
            type=semver,pattern={{major}}.{{minor}}-${{ matrix.arch }}

      - name: Build and push app image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.app
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=${{ matrix.arch }}
          cache-to: type=gha,mode=max,scope=${{ matrix.arch }}
          platforms: ${{ matrix.platform }}

      - name: Extract metadata for DB image
        id: meta-db
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-db
          tags: |
            type=raw,value=latest-${{ matrix.arch }}
            type=semver,pattern={{version}}-${{ matrix.arch }}
            type=semver,pattern={{major}}.{{minor}}-${{ matrix.arch }}

      - name: Build and push db image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.db
          push: true
          tags: ${{ steps.meta-db.outputs.tags }}
          labels: ${{ steps.meta-db.outputs.labels }}
          cache-from: type=gha,scope=${{ matrix.arch }}
          cache-to: type=gha,mode=max,scope=${{ matrix.arch }}
          platforms: ${{ matrix.platform }}

  # マルチアーキテクチャマニフェストの作成
  create-manifest:
    needs: build-and-push
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract version
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
            echo "version=$VERSION" >> $GITHUB_OUTPUT
          fi

      - name: Create app manifest
        run: |
          .github/scripts/create-manifest.sh \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}" \
            "app" \
            "${{ steps.version.outputs.version }}"

      - name: Create db manifest
        run: |
          .github/scripts/create-manifest.sh \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}" \
            "db" \
            "${{ steps.version.outputs.version }}"
```

タグがプッシュされたらAMD/ARM向けにRustビルドを行ってDockerイメージを作成、
それをGitHub Container Registoryに登録するということをやっています。
これによってdocker pullで `ghcr.io/GitHubユーザー名/リポジトリ名` を指定すればコンテナーを取得して利用できます。


## さいごに
