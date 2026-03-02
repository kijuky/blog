https://qiita.com/kijuky/items/4034f5ed49507872e4a7

---

Heroku は無料で実行コンテナとデータベースがついてくるクラウドサービスです。言語別のサンプルデモアプリもあり、Scala+PlayFrameworkもあるのですが、バージョンが古いのでアップグレードしてみましょう。

# アップデート手順

## 巨人の肩に乗る

[元のGitHub](https://github.com/heroku/scala-getting-started)の[フォーク一覧](https://github.com/heroku/scala-getting-started/network)を調べると、[play-2.8を試行している](https://github.com/fabjan/scala-getting-started/tree/play-2.8)人がいるので、これを足場として作業しましょう。

## 各種ライブラリをバージョンアップする

| ライブラリ | 変更前 | 変更後 | 備考 |
|:-:|:-:|:-:|:-|
| Java | 1.8 | 11.0.16 | `system.properties` に設定 |
| Scala | 2.11.7 | 2.13.8 | `build.sbt` に設定 |
| sbt | 0.13.11 | 1.7.1 | `project/build.properties` に設定 |
| Play | 2.4.3 | 2.8.16 | `project/plugin.sbt` に設定 |

## アプリケーションシークレットの追加

https://www.playframework.com/documentation/latest/ApplicationSecret

```hocon:application.conf
application.secret="change-me"
```

↓

```hocon:application.conf
play.http.secret.key=${?APPLICATION_SECRET}
```

ローカル開発では適当なシークレットを追加しておきましょう。

```shell:.env
APPLICATION_SECRET=QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n
```

本番用には、まずシークレットを生成して

```console
$ sbt playGenerateSecret
...
Generated new secret: <<生成されたシークレット>>
...
```

構成変数に設定しておきましょう。

```console
$ heroku config:set 'APPLICATION_SECRET=<<生成されたシークレット>>'
```

## 許可ホストを設定する

https://qiita.com/s-katsumata/items/6e16dbadbf4d5516010f

```hocon:application.conf
play.filters.hosts.allowed = ["."]
```

## プロパティ名の変更に対応する

https://www.playframework.com/documentation/latest/Migration24#Configuration-changes

```hocon:application.conf
application.langs="en"
```

↓

```hocon:application.conf
play.i18n.langs=["en"]
```

## ローカルデータベースに postgresql を使う

足場に使ったコミットでは h2 を使うようになっていますが、本番と構成が同じ方が接続情報などを流用できるため、postgresql を使うようにしてみましょう。

```bash:.envrc
export POSTGRES_PASSWORD=password
export DATABASE_URL=postgres://postgres:${POSTGRES_PASSWORD}@localhost:5432/postgres
```

```docker-compose.yml
version: "3"

services:
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    ports:
      - "5432:5432"
    restart: always
```

```hocon:application.conf
db.default.driver=org.postgresql.Driver
db.default.url=${?DATABASE_URL}
```

```build.sbt
libraryDependencies ++= Seq(
  jdbc,
  caffeine,
  ws,
  guice,
  "org.postgresql" % "postgresql" % "42.3.6"
)
```

ローカル開発する際はdbを立ち上げておきましょう

```console
$ docker compose up
```

## resolvers の削除

https://www.scala-sbt.org/1.x/docs/Resolvers.html

resolvers に設定されている Typesafe repository はデフォルトで入っているので不要です。

## Procfile の修正

console が起動しなくなっているので、削除しておきます。

## heroku stack のアップグレード

https://devcenter.heroku.com/articles/upgrading-to-the-latest-stack

デフォルトだと 20 で作成されるので、 22 にアップグレードします。

```console
$ heroku stack:set heroku-22
```

# デプロイ

ここまでできたらデプロイしておきます。お疲れ様でした。

```console
$ git push heroku master
$ heroku open
```

# まとめ

heroku でも最新の play が動くことが確認できました。良いですね。
