https://qiita.com/kijuky/items/89accbeaffdb39457dd4

---

ごきげんよう。皆さんはWebアプリ開発の際に環境構築ってめんどくさいなぁと思ったこと、ありませんか？私はあります。もし「そんなこと経験したことないなぁ」という方であれば、レガシーソフトウェア改善ガイドの第7章を読みましょう。

https://www.shoeisha.co.jp/book/detail/9784798145143

かいつまんで説明すると、散らばったビルドスクリプトや環境構築手順書が腐っている上に、ひとつひとつコマンドを叩いたり、DBアカウントの申請が必要だったりと、レガシーソフトウェアはその開発環境の構築すらも困難になっていることがあります。それを [vagrant](https://www.vagrantup.com/) + [ansible](https://github.com/ansible/ansible) で自動化し、`「仕事を始める日」（テイク2）`では、`vagrant up`を起動した後にコーヒーマシンの使い方を調べる余裕までできちゃうぜ、っていうストーリーです。

これ自体はすごく素敵なことなのですが、[最近の macOS だと vagrant が動かない](https://qiita.com/syota_19910612bscplog/items/ee7e06cd23603fa529dd)ことがあったりして、そもそもの vagrant 環境構築が辛くなってきました。ので、vagrantの代わりに [docker](https://docs.docker.jp/index.html) を使って幸せになりましょう、というのがこの記事の趣旨です。

# ローカル開発環境自動化、とは？

vagrantもansibleも聞いたことない、っていう、ちょっと昔の私からすると「ローカル開発環境自動化ナニソレ？おいしいの？」って感じだったので、改めてそのメリットを整理します。ローカル開発環境自動化を行うことで、以下のような開発体験が得られます。

- dockerが入っているPCであれば、あなたのアプリを動作させることができます。
- コマンド1つで、DBの立ち上げ、複数のアプリの立ち上げだって可能です。
- ソースコードはdockerコンテナと共有できるので、コンテナ側でホットリロード状態で起動していれば、コードの変更を検知して自動で再コンパイルさせることもできます。

ただし、現状dockerコンテナは基本的にはlinuxなので、Xcode Toolchainを要求するようなmacOS/iOS開発だと、恩恵が得にくいかもしれません...

# 基本戦略

ここでは、 [PlayFramework](https://www.playframework.com/) を例に、ローカル開発環境を自動化してみましょう。手始めにこんな `docker-compose.yml` を用意します。

```yml:docker-compose.yml
version: '3'

services:
  app:
    build:
      context: .
      dockerfile: ./docker/local/app/Dockerfile
    command: sbt run
    environment:
      - SBT_OPTS=-Dsbt.boot.directory=.sbt -Dsbt.coursier.home=.coursier
    ports:
      - "9000:9000"
    volumes:
      - .:/app
```

```dockerfile:docker/local/app/Dockerfile
FROM hseeberger/scala-sbt:11.0.14.1_1.6.2_2.13.8

# locale
RUN apt-get update && apt-get install -y locales
RUN sed -i -E 's/# (ja_JP.UTF-8)/\1/' /etc/locale.gen && locale-gen
ENV LC_ALL=ja_JP.UTF-8 TZ=Asia/Tokyo

# volumes
WORKDIR /app
```

これを `docker compose up` すると、

- 日本語ロケールで
- sbt/coursierはホスト側にキャッシュして（2回目以降はキャッシュからjarを解決できる）
- ソースコードはコンテナと共有し
- ホットリロード状態で起動

したPlayアプリが `http://localhost:9000`　で立ち上がります。あとはソースコードを変更したら勝手にリコンパイルされるので、ブラウザで変更を確認できます。便利でしょ？

ここからは、これを前提として、細かい tips を紹介します。

## 特定のディレクトリはホストと共有させない

例えば`node_modules`のように、OSに依存したバイナリを含むディレクトリをmacOSホスト+Linuxコンテナで共有すると、ホストかコンテナのどちらかでしか起動できなくなります。そういうめんどくさい環境は作りたくないですよね？

全てのソースコードを共有しましたが、一部のフォルダだけ共有したくない場合は`docker-compose.yml`の`volumes`にそのフォルダ名を書くと、共有から外れます。

```yml:docker-compose.yml
    volumes:
      - .:/app
      - /app/node_modules
```

ただ、こうしてしまうとコンテナ側には `node_modules` がないので、`npm ci`が失敗してしまいます。なので、このアプリの場合はDockerfileの中でnode_modulesを構築してしまいます。

```dockerfile:docker/local/app/Dockerfile
# (上の続き)

# install node packages
COPY package*.json /app/
RUN npm ci
```

Dockerfileビルド時はappディレクトリには何もマウントされてないので、`npm ci`に必要な`package*.json`をコンテナにコピーしておきます。

https://zenn.dev/foolishell/articles/3d327557af3554

## https でアクセスしたい

アプリがなんらかの外部リソースにアクセスする際にhttpsを要求すること、あると思います。これは [https-portal](https://github.com/SteveLTN/https-portal) を使いましょう。

```yml:docker-compose.yml
services:
  ...

  https-portal:
    image: steveltn/https-portal:1
    ports:
      - "80:80"
      - "443:443"
    links:
      - app
    restart: always
    environment:
      STAGE: local
      DOMAINS: "localhost.hoge.jp -> http://app:9000"
    volumes:
      - ./.ssl_certs:/var/lib/https-portal
```

```conf:/etc/hosts
127.0.0.1       localhost.hoge.jp # 追加
```

`STAGE: local`という設定は自己証明書を自動で作成する設定になります。作成された証明書は`.ssl_certs`に配置されます（キャッシュ用）。これで `https://localhost.hoge.jp` でアプリにアクセスすることができます。

https://genchan.net/it/virtualization/docker/331/

## 一緒にDBも立ち上げたい

例えば [postgres](https://www.postgresql.org/) だとこんな感じ。

```yml:docker-compose.yml
services:
  db:
    image: postgres:10.3
    environment:
      - POSTGRES_DB=${POSTGRES_DB_NAME:-app}
      - POSTGRES_USER=${POSTGRES_DB_USER:-app}
      - POSTGRES_PASSWORD=${POSTGRES_DB_PASSWORD:-app}
    ports:
      - "5432:5432"

  app:
    ...
    
    environment:
      - POSTGRES_DB_URL=jdbc:postgresql://db:5432/${POSTGRES_DB_NAME:-app}
      - POSTGRES_DB_USER=${POSTGRES_DB_USER:-app}
      - POSTGRES_DB_PASSWORD=${POSTGRES_DB_PASSWORD:-app}
    links:
      - db
```

postgresでは「デフォルトのDB名(`POSTGRES_DB`)」「ユーザー名(`POSTGRES_USER`)」「パスワード(`POSTGRES_PASSWORD`)」くらいは設定に含めておくと良さげ。使う側はJDBC用のURLとユーザー名、パスワードを環境変数として設定しておく感じ。`links`に`db`サービスを指定することで、`app`は`db`の立ち上げ後に立ち上がり、かつ`db`をホスト名として利用することができます（上でJDBC用のURLに`db:5432`として利用している）。

通常は（Playみたいなリッチなフレームワーク使ってると）フレームワークのマイグレーションツール使うのが楽ですが、仮に使ってなかったとしても、そういうDockerfile用意して`links`に`db`を指定してあげるといい感じです。

```Dockerfile:docker/local/db_setup/Dockerfile
FROM postgres:10.3

# install flyway
# https://flywaydb.org/documentation/usage/commandline/#-linux
RUN apt-get update && apt-get install -y wget
RUN VERSION=8.5.7 && wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${VERSION}/flyway-commandline-${VERSION}-linux-x64.tar.gz | tar xvz && ln -s `pwd`/flyway-${VERSION}/flyway /usr/local/bin

# volumes
WORKDIR /app

# setup
ENTRYPOINT ["/app/docker/local/db_setup/entrypoint.sh"]
```

```sh:docker/local/db_setup/entrypoint.sh
#!/bin/sh
set -euC

# なんかいい感じのマイグレーション処理
flyway \
  -user="$POSTGRES_DB_USER" \
  -password="$POSTGRES_DB_PASSWORD" \
  -url="$POSTGRES_DB_URL" \
  -locations=filesystem:db/migration/ \ # db/migration フォルダにいい感じに作っとく
  migrate
```

```yml:docker-compose.yml
services:
  ...

  db_setup:
    build:
      context: .
      dockerfile: docker/local/db_setup/Dockerfile
    environment:
      - POSTGRES_DB_URL=jdbc:postgresql://db:5432/${POSTGRES_DB_NAME:-app}
      - POSTGRES_DB_USER=${POSTGRES_DB_USER:-app}
      - POSTGRES_DB_PASSWORD=${POSTGRES_DB_PASSWORD:-app}
    links:
      - db
    volumes:
      - .:/app
```

## 独自ホストを解決したい

例えば、別でホストされているアプリと接続する必要があり、開発環境では、開発環境からしか繋げないような場所にあるオンプレのサーバーに接続したいとか、まあ、あると思います。そんな時は `extra_hosts` を使います。


```yml:docker-compose.yml
services:
  app:
    ...
    
    extra_hosts:
      - qa1:192.168.0.101
      - qa2:192.168.0.102
```

これで`app`サービス内では`qa1`ホストや`qa2`ホストにアクセスできます。

# まとめ

私はこれまでDockerの主な用途はクラウド上にいい感じにホストしてうまうま、という世界しか知らなかったので、このようにローカル開発環境自動化としても利用できるのはちょっと目から鱗でした。今後も便利に使っていきたいと思います。
