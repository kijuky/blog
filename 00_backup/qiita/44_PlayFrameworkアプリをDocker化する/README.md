https://qiita.com/kijuky/items/a54e94b293e08ff3b839

---

最近はWebアプリをDocker化してクラウド上に展開することが増えてきていると思います。それを受けてPlayFramework製のアプリをDocker化する記事もちらほら見かけます。一方で、それらの記事でSBT Native Packagerを使ってDocker化しているのをあまり見かけなかったので、この記事ではその紹介をします。

# SBT Native Packager とは

https://github.com/sbt/sbt-native-packager

(Playに限らず)Scala製のアプリケーションを様々なパッケージ形式に変換するプラグインです。jar はもちろん、rpm や msi などのインストーラー形式だったり、GraalVM によるネイティブ形式だったり、今回紹介する Docker 形式だったりに対応しています。

Playには、バージョンは多少古いですがこのプラグインを内蔵しており、Playアプリケーションにおいては追加のプラグインなしにDocker化を行うことができます。

## Docker 化

早速Docker化をみてみましょう。ここでは scala-seed プロジェクトをDocker化してみます。作られたプロジェクトの build.sbt をこんな感じにします。

```scala:build.sbt
name := """test"""
organization := "com.example"

version := "1.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)
  .enablePlugins(DockerPlugin)
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11"
  )

scalaVersion := "2.13.8"

libraryDependencies += guice
libraryDependencies += "org.scalatestplus.play" %% "scalatestplus-play" % "5.0.0" % Test

// Adds additional packages into Twirl
//TwirlKeys.templateImports += "com.example.controllers._"

// Adds additional packages into conf/routes
// play.sbt.routes.RoutesKeys.routesImport += "com.example.binders._"
```

https://sbt-native-packager.readthedocs.io/en/stable/recipes/play.html#application-configuration

驚くべきことに、あなたはDockerのベースイメージを決めるだけで、Dockerfileを書かなくて良いのです。

さて、とりあえず起動してみる場合は、secret を追加しておきましょう。

https://www.playframework.com/documentation/2.8.x/Deploying#The-application-secret

```hocon:conf/application.conf
# https://www.playframework.com/documentation/latest/Configuration
play.http.secret.key="QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n"
```

では、起動してみます。

```console
$ sbt Docker/publishLocal
...(省略)
[error] #14 naming to docker.io/library/test:1.0-SNAPSHOT done
...(省略)
$ docker run -it -p 9000:9000 test:1.0-SNAPSHOT
...(省略）
2022-08-05 23:02:53 INFO  play.core.server.AkkaHttpServer  Listening for HTTP on /0.0.0.0:9000
```

起動したっぽいですね。 http://localhost:9000 にアクセスしてみましょう。

`Welcome to Play!` が見れましたか？これを前提に、以下では様々なTipsを紹介します。

## どんなDockerfileが作成されるの？

Dockerfile 書いてない、書かなくて良いと言われたって、時には生成される Dockerfile をみてみたいものです。そんな時は Docker/stage を実行し、`target/docker/stage/Dockerfile` をみてみましょう。

```console
$ sbt Docker/stage
...(省略）
$ cat target/docker/stage/Dockerfile
```

標準で[マルチステージビルド](https://matsuand.github.io/docs.docker.jp.onthefly/develop/develop-images/multistage-build/)されていて、[デーモンユーザー(demiourgos728)で起動](https://matsuand.github.io/docs.docker.jp.onthefly/storage/bind-mounts/)しているのがわかります。デーモンユーザーは書き込み権限がないので、コンテナの中身をぶっ壊さないのが保証されてて良いですね。

## `./logs/application.log (No such file or directory)` が出る

上との関連ですが、デーモンユーザーはコンテナへの書き込み権限がないので、そもそも logs フォルダを作れないんですね。ログはコンテナの外に記録されるようにしましょう。

https://matsuand.github.io/docs.docker.jp.onthefly/storage/bind-mounts/

```console
$ docker run -it -p 9000:9000 -v "$(pwd)/logs:/opt/docker/logs" test:1.0-SNAPSHOT
```

クラウドにあげる際も、似たような形でマウントすると思います。マウント先のデフォルトは `/opt/docker/logs` と覚えておけば良いでしょう。

## ビルド時に `[1] There are no exposed ports for your docker image` が出る

サジェストにあるように、build.sbt に expose ports を設定してあげましょう。

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11",
    dockerExposedPorts ++= Seq(9000)  // <- これ
  )
```

これにより Dockerfile に `EXPOSE 9000` が追加されます。

## タイムゾーンを `Asia/Tokyo` にしたい

`dockerEnvVars` を使います。これ以外にも必要な環境変数を追加できます。

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11",
    dockerExposedPorts ++= Seq(9000),
    dockerEnvVars := Map("TZ" -> "Asia/Tokyo") // <- これ
  )
```

## 本番用のコンフィグファイルで起動したい。

エントリーポイントにて引数を追加します。

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11",
    dockerExposedPorts ++= Seq(9000),
    dockerEnvVars := Map("TZ" -> "Asia/Tokyo"),
    dockerEntrypoint := Seq("bin/test", s"-Dconfig.resource=${sys.env.getOrElse("ENV", "")}.conf") // <- これ
  )
```

この例では `conf/${ENV}.conf` ファイルが指定されます。他にも好きな引数を `dockerEntrypoint` に追加できます。

## カスタムエントリーポイントを使いたい

Playで作られるエントリーポイントではなく、最初にちょっとした設定を施してアプリを起動したい、あると思います。そんな時は `dist/bin/<<エントリーポイント>>` を作成します。

```bash:dist/bin/entrypoint.sh
#!/bin/bash

echo "custom entrypoint!"

bin/test
```

このファイルを `dockerEntrypoint` とします。

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11",
    dockerExposedPorts ++= Seq(9000),
    dockerEnvVars := Map("TZ" -> "Asia/Tokyo"),
    dockerEntrypoint := Seq("bin/entrypoint.sh") // <- これ
  )
```

Playの規約で、[dist ディレクトリにあるファイルは全てパッケージの中にコピーされます](https://www.playframework.com/documentation/2.8.x/Deploying#Including-additional-files-in-your-distribution)。つまり `dist/bin/entrypoint.sh` は（`WORKDIR` が `/opt/docker` の場合） `/opt/docker/bin/entrypoint.sh` にコピーされます。次に、[SBT Native Packagerの規約で `/opt/docker/bin/*` は実行可能ファイルとしてマークされます](https://sbt-native-packager.readthedocs.io/en/stable/formats/docker.html#file-permission)。カレントディレクトリは `/opt/docker` なので `bin/プロジェクト名` でアプリが起動します。

## クラウドへプッシュしたい

`dockerRepository`, `Docker / packageName`, `Docker / version` を設定します。`packageName` と `version` は `ThisBuild` と同じ値なら変更する必要はありません。 `dockerRepository` は `Option[String]` であることに注意します。

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "eclipse-temurin:11",
    dockerExposedPorts ++= Seq(9000),
    dockerEnvVars := Map("TZ" -> "Asia/Tokyo"),
    dockerEntrypoint := Seq("bin/entrypoint.sh"),
    dockerRepository := Some(s"${sys.env.getOrElse("AWS_ACCOUNT", "")}.dkr.ecr.${sys.env.getOrElse("AWS_REGION", "ap-northeast-1")}.amazonaws.com") // <- これ
  )
```

ログインが必要な場合、`sbt Docker/publish` の前にログインしていればOKです。

```
$ aws ecr get-login-password --region "${AWS_REGION}" | docker login --username AWS --password-stdin "${AWS_ACCOUNT}".dkr.ecr."${AWS_REGION}".amazonaws.com
$ sbt Docker/publish
```

出てきた環境変数は適宜読み替えてください。

## ベースイメージに Amazon Corretto を使用したい。

https://docs.aws.amazon.com/ja_jp/corretto/latest/corretto-11-ug/docker-install.html

Amazon Corretto のデフォルトベースイメージが Amazon Linux 2 であるため、`useradd` がなく、デーモンユーザーの追加ができずにビルドに失敗します。[デーモンユーザーでの起動を諦める](https://sbt-native-packager.readthedocs.io/en/stable/formats/docker.html#daemon-user)か、なんとかして `useradd` をインストールする必要があります。

https://blog.pinkumohikan.com/entry/install-useradd-command-to-amazon-linux2

```scala:build.sbt
  ...(省略）...
  .settings(
    Universal / javaOptions ++= Seq("-Dpidfile.path=/dev/null"),
    dockerBaseImage := "amazoncorretto:11",
    dockerExposedPorts ++= Seq(9000),
    dockerEnvVars := Map("TZ" -> "Asia/Tokyo"),
    dockerEntrypoint := Seq("bin/entrypoint.sh"),
    dockerRepository := Some(s"${sys.env.getOrElse("AWS_ACCOUNT", "")}.dkr.ecr.${sys.env.getOrElse("AWS_REGION", "ap-northeast-1")}.amazonaws.com"),
    // ↓これ
    dockerCommands := {
      // install useradd
      import com.typesafe.sbt.packager.docker._
      dockerCommands.value.foldLeft[Seq[CmdLike]](Nil) { (commands, command) =>
        commands ++ {
          command match {
            case Cmd("USER", "root") =>
              Seq(
                command,
                Cmd(
                  "RUN",
                  "yum install -y shadow-utils && rm -rf /var/cache/yum/* && yum clean all"
                )
              )
            case _ =>
              Seq(command)
          }
        }
      }
    }
  )
```

`RUN yum install -y shadow-utils && rm -rf /var/cache/yum/* && yum clean all` を追加したいだけなのにこの記述量は萎えますね... Amazon Corretto は alpine ベースのイメージもありますが、そっちは ash だったりするので、[AshScriptSupport が必要](https://sbt-native-packager.readthedocs.io/en/stable/formats/docker.html#busybox-ash-support)だったりします。ただ、 [is_cygwin がなかったりする](https://github.com/playframework/playframework/issues/8282)ので、 ubuntsu や bullseye 的なベースイメージに Amazon Corretto をインストールしたカスタムイメージを独自に用意する方が良いかもしれません。

# まとめ

Playは標準でSBT Native Packagerを同梱しているので、Docker化やそのイメージのプッシュは build.sbt だけで完結します。便利ですね！！

それではみなさん、良きPlay/Scala/Dockerライフを！
