https://qiita.com/kijuky/items/398b20f968d98408e596

---

この記事では、2023年における、PlayFrameworkを使ったアプリケーションの構築方法の紹介をします。作成するアプリケーションは、いわゆるHelloWorld的なもので、ブラウザでアクセスすれば、何らかレスポンスが返ってくるものを作ります。

それでは、どうぞ。

# 事前準備

まずは[coursier](https://get-coursier.io/)をセットアップします。セットアップの詳細はこちらの記事を確認してください。

https://www.scala-lang.org/download/

https://blog.3qe.us/entry/scala-2023

# 爆速！PlayFrameworkアプリ作成

coursierでインストールされるsbtには`sbt new`というコマンドがあり、これを使うとPlayFrameworkのひな型を作ってくれます。便利ですね。

https://github.com/sbt/sbt/releases/tag/v1.9.0#:~:text=sbt%20new%2C%20a%20text%2Dbased%20adventure

```shell
> sbt new

Welcome to sbt new!
Here are some templates to get started:
 a) scala/toolkit.local               - Scala Toolkit (beta) by Scala Center and VirtusLab
 b) typelevel/toolkit.local           - Toolkit to start building Typelevel apps
 c) sbt/cross-platform.local          - A cross-JVM/JS/Native project
 d) scala/scala3.g8                   - Scala 3 seed template
 e) scala/scala-seed.g8               - Scala 2 seed template
 f) playframework/play-scala-seed.g8  - A Play project in Scala
 g) playframework/play-java-seed.g8   - A Play project in Java
 i) softwaremill/tapir.g8             - A tapir project using Netty
 m) scala-js/vite.g8                  - A Scala.JS + Vite project
 n) holdenk/sparkProjectTemplate.g8   - A Scala Spark project
 o) spotify/scio.g8                   - A Scio project
 p) disneystreaming/smithy4s.g8       - A Smithy4s project
 q) quit
Select a template:
```

`f` を押します。

```shell
Select a template: f
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
This template generates a Play Scala project

name [play-scala-seed]:
organization [com.example]:
play_version [2.8.20]:
scala_version [2.13.12]:

Template applied in ~/play-scala-seed
```

`This template generates a Play Scala project` 以下は[g8](https://www.foundweekends.org/giter8/ja/index.html)の出力になります。プロジェクト名（`name`）、パッケージ名（`organization`）、Playバージョン（`play_version`）、Scalaバージョン（`scala_version`）を入力してエンターを押します。

テンプレートができたらcdして、JVMが11であることを確認したのち、`sbt run`します。

```shell
> cd play-scala-seed
> sbt run
[info] welcome to sbt 1.7.2 (Amazon.com Inc. Java 11.0.20.1)
[info] loading global plugins from ~\.sbt\1.0\plugins
[info] loading settings for project play-scala-seed-build from plugins.sbt ...
[info] loading project definition from ~\play-scala-seed2\project
[info] loading settings for project root from build.sbt ...
[info]   __              __
[info]   \ \     ____   / /____ _ __  __
[info]    \ \   / __ \ / // __ `// / / /
[info]    / /  / /_/ // // /_/ // /_/ /
[info]   /_/  / .___//_/ \__,_/ \__, /
[info]       /_/               /____/
[info]
[info] Version 2.8.20 running Java 11.0.20.1
[info]
[info] Play is run entirely by the community. Please consider contributing and/or donating:
[info] https://www.playframework.com/sponsors
[info]

--- (Running the application, auto-reloading is enabled) ---

[info] p.c.s.AkkaHttpServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9000

(Server started, use Enter to stop and go back to the console...)
```

ブラウザから http://localhost:9000 を実行すると、Playが起動します。

# まとめ

`sbt new`でPlayFrameworkのひな型が簡単に作成できることを紹介しました。ほかにもいろんなひな型があるようなので、Scalaでどんなプロダクトを作れるのか、一通り見てみるのも面白そうですね。

今回は以上です。皆さん、良きScalaライフを！
