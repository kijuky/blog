https://qiita.com/kijuky/items/5d3a2c78e4ad22e0d2ff

---

[Scala Steward](https://github.com/scala-steward-org/scala-steward)とは、依存ライブラリのバージョンアップ確認をして、セマンティックバージョニング的にバージョンアップが可能であればプルリクエスト/マージリクエストを作成してくれるサービスです。似たサービスに[Renovate](https://www.mend.io/renovate/)がありますが、Scala StewardはScalaに特化したバージョンアップ確認サービスです。

この記事では、Scala StewardをPlayFrameworkサービスに適用した際に追加した設定を紹介します。

# 背景：sbtによる依存ライブラリの解決について

sbt1.5から、依存ライブラリのバージョン範囲がより厳密になりました。全ての依存ライブラリが必要とするライブラリはメジャーバージョンが揃っている必要があります（例外もあります）。これにより、[scala-xml](https://github.com/scala/scala-xml)や[scala-parser-combinators](https://github.com/scala/scala-parser-combinators)のライブラリの違いでビルドができないことがあります。

他にも[SLF4J](https://www.slf4j.org/)/[Logback](https://logback.qos.ch/)や[jackson-databaind](https://github.com/FasterXML/jackson-databind)などの、コンパイル時にはわからないがランタイムでエラーを起こす厄介者もいます。

# おさらい：Scala Stewardのignore設定

この様な状況下で、単にScala Stewardを導入してしまうと、依存ライブラリのバージョンを上げると、ビルドエラーだったりランタイムエラーが発生してしまったりします。そこで、実際には特定のバージョンを無視したり、パッチバージョンだけは取得したり（＝ピン留め）する必要が出てきます。Scala Stewardでは[`.scala-steward.conf`](https://github.com/scala-steward-org/scala-steward/blob/main/docs/repo-specific-configuration.md)というファイルを置くことで、それを実現できます。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.scala-sbt", artifactId = "sbt", version = "1.7.2" }
]
```

このファイルは（多分）[HOCON](https://github.com/lightbend/config/blob/main/HOCON.md)形式で書かれます。Scala使いにはお馴染みかと思いますが、「初めて見たよ！」という方は、コメントが書けるJSONとしてとりあえず認識してもらえると良いです。

`.scala-steward.conf`には色々な機能があるのですが、はじめに`updates.pin`を使えるようになると便利です。これにより、依存ライブラリの「ピン留め」が利用できる様になります。上に挙げた例は、sbtのバージョンを1.7.2で固定させて、それ以上のバージョンアップを行わない、という指定になります。

なお、`.scala-steward.conf`には「ピン留め」以外にもいろんな機能があります。これらは公式ドキュメントを読むのも良いですが、既存プロジェクトの設定を見てみるのが一番手っ取り早かったりもします。

# 実践：PlayFrameworkにおけるScala Stewardのignore設定

Play2.8は約4年前にリリースされました。そしてPlayはメジャーバージョンの範囲では依存ライブラリのバージョンアップをしない傾向にあります。つまり、依存ライブラリは基本的に4年前のものを採用しており、そのままScala Stewardで依存ライブラリをバージョンアップしてしまうと、Play自体の依存ライブラリとバッティングしてしまって、ビルドエラーになってしまいます（ちなみに、これがplayのバージョンアップの醍醐味だったりもします）。

そこで、Play2.8時点で設定した方が良いignore設定を紹介します。

## Play2.8

素のPlay2.8でも設定しておいた方が良いのがsbtのピン留めです。sbtは1.7.3以降からscala-xmlのバージョンとして2系を採用しており、Play2.8とバッティングします。

また、直近でPlay2.9の準備のため、依存ライブラリのバージョンアップが行われました。これらも固定化した方が良いでしょう。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.scala-sbt", artifactId = "sbt", version = "1.7.2" },
  { groupId = "com.typesafe.sbt", artifactId = "sbt-twirl", version = "1.5." },
  { groupId = "com.typesafe.play", artifactId = "play-json", version = "2.9." },
  { groupId = "com.typesafe.play", artifactId = "play-json-joda", version = "2.9." }
]
```

## Play2.8 + [Scoverage](https://github.com/scoverage/sbt-scoverage)

テストカバレッジを取る場合、scalaのバージョンもピン留めする必要があります。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.scoverage", artifactId = "sbt-scoverage" version = "1.9.3" },
  { groupId = "org.scala-lang", artifactId = "scala-compiler", version = "2.13.8" },
  { groupId = "org.scala-lang", artifactId = "scala-library", version = "2.13.8" },
  { groupId = "org.scala-lang", artifactId = "scala-reflect", version = "2.13.8" }
]
```

## Play2.8 + SLF4J + Logback

SLF4Jは2系を選択できず、2系に依存するLogbackも指定できません。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.slf4j", artifactId = "slf4j-api", version = "1.7." },
  { groupId = "org.slf4j", artifactId = "jcl-over-slf4j", version = "1.7." },
  { groupId = "org.slf4j", artifactId = "log4j-over-slf4j", version = "1.7." },
  { groupId = "org.slf4j", artifactId = "jul-to-slf4j", version = "1.7." },
  { groupId = "ch.qos.logback", artifactId = "logback-classic", version = "1.2." }
]
```

## Play2.8 + [ScalikeJDBC](http://scalikejdbc.org/)

jackson-databindのバージョンが異なるため、ScalaikeJDBCの最新バージョンを設定できません。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.scalikejdbc", artifactId = "scalikejdbc", version = "3.5.0" },
  { groupId = "org.scalikejdbc", artifactId = "scalikejdbc-config", version = "3.5.0" },
  { groupId = "org.scalikejdbc", artifactId = "scalikejdbc-play-fixture", version = "2.8.0-scalikejdbc-3.5" },
  { groupId = "org.scalikejdbc", artifactId = "scalikejdbc-play-initializer", version = "2.8.0-scalikejdbc-3.5" },
  { groupId = "org.scalikejdbc", artifactId = "scalikejdbc-syntax-support-macro", version = "3.5.0" }
]
```

## Play2.8 + [flyway-play](https://github.com/flyway/flyway-play)

scala-xmlのバージョンが異なるため、特定のバージョンより上に上げられません。


```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.flywaydb", artifactId = "flyway-play", version = "7.25.0" }
]
```

## Play2.9 + flyway-play

Play2.9であっても、最新版にはバージョンアップできません。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.flywaydb", artifactId = "flyway-play", version = "7.29.0" }
]
```

## Play2.8 + [h2](https://www.h2database.com/html/main.html)

テストに使うDBとしてh2を使う場合、最新に上げてしまうと警告が出まくるのでピン留めします。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "com.h2database", artifactId = "h2", version = "2.1.214" }
]
```

## Play2.8 + servlet-api

サーブレットAPIに依存する場合、jackson-databindのバージョンを揃える必要があります。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "javax.servlet", artifactId = "javax.servlet-api", version = "3.1.0" }
]
```

## Play2.8 + [Scalate](https://scalate.github.io/scalate/)

scala-parser-combinatorsのバージョンを揃える必要があります。

```hocon:.scala-steward.conf
updates.pin = [
  { groupId = "org.scalatra.scalate", artifactId = "scalate-core", version = "1.9.6" }
]
```

# 補足：scala-xmlについて

scala-xmlについては1系と2系で実際は互換性があるとかないとか。どうしても新しいバージョンを入れたい場合は、依存ライブラリ側のscala-xmlを外して1系に揃える、という手もありかもしれません。ただ、Play2.9がscala-xmlの2系に揃ったので、Play2.9にバージョンアップするのが総合的に見て良い選択肢かもしれません。

# まとめ

PlayFrameworkにおけるScala Stewardのignore設定を紹介しました。不必要なバージョンアップ通知がなくなることで、依存ライブラリのバージョンアップの信頼性が上がって、より安全に依存ライブラリのバージョンアップができることでしょう。

以上です。皆さんも良きScalaライフを！
