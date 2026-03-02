https://qiita.com/kijuky/items/21be8d13a89a4b835ad4

---

この記事では Scala で Jar ライブラリやネイティブライブラリを作成する方法について解説します。

# 背景

Java(javac) は後方互換性なバイナリを生成するため、広く使われることが予想されるライブラリは、LTS で一番低いバージョンでビルドしておくと、多くの方が利用できるでしょう。しかしながら、Scala 2.x は(semberの)マイナーバージョン間で非互換なバイナリを生成するため、各マイナーバージョンでライブラリを作成する必要があります。sbt には、複数の Scala のバージョンをビルドする機能が備わっています。

# Jar ライブラリを作る

プロジェクトに `crossScalaVersions` を与えることで、複数の Scala バージョンをビルドさせることができます。

```scala:build.sbt
ThisBuild / name := "my-library"

lazy val scala3 = "3.1.1"
lazy val scala213 = "2.13.8"
lazy val scala212 = "2.12.15"
lazy val scala211 = "2.11.12"
lazy val supportedScalaVersions = List(scala3, scala213, scala212, scala211)

lazy val root = (project in file("."))
  .settings(
    scalaVersion := scala213,

    // クロスビルド設定
    crossScalaVersions := supportedScalaVersions,

    // maven リポジトリ用
    organization := "org.github.kijuky",
    publishTo := {
      if (isSnapshot.value) {
        Some("snapshots" at ...) // snapshot は自由に配置できるように。
      } else {
        sys.env.get("PUBLISH_URL").map("release".at) // release は CI で定義された環境変数(PUBLISH_URL) から配置場所を決定。
      }
    }
  )
```

この `crossScalaVersions` が与えられているとき、コマンドの先頭に `+` を与えると、 `scalaVersion` をそのリストのバージョンにして foreach してくれます。便利ですね。

```shell
$ sbt +compile
```

これで Scala 3.x, Scala 2.13.x, Scala 2.12.x, Scala 2.11.x のコンパイルが行われます。

テストも、

```shell
$ sbt +test
```

maven リポジトリへの公開も、同じようにできます。

```shell
$ sbt +publish # PUBLISH_URL が必要
```

## フォルダ構成

Scala のバージョンが異なる場合、ソースコードの書き方も変わってきたりします。例えば、Scala 2.12 から Scala 2.13 では、Java のコレクションライブラリから Scala のコレクションライブラリに変換するパッケージが変わっていたりします。Scala 3 に至ってはオプショナルブレースによってかなり表記が変わります。つまり、Scala はバージョン間でソースコード互換性が保たれていないのです。

このような場合、sbt では scala のバージョンの違いでフォルダを切り替えることができます。

```
root
├ project
│ ├ build.properties
│ └ plugins.sbt
├ src
│ ├ main
│ │ ├ scala      # 全ての scala バージョンでコンパイルされる
│ │ ├ scala-2.11 # scala 2.11.x の時だけコンパイルされる
│ │ ├ scala-2.12 # scala 2.12.x の時だけコンパイルされる
│ │ ├ scala-2.13 # scala 2.13.x の時だけコンパイルされる
│ │ └ scala-3    # scala 3.x の時だけコンパイルされる
│ └ test         # test でも同様のフォルダ構成を利用できる
│   ├ scala
│   ├ scala-2.11
│   ├ scala-2.12
│   ├ scala-2.13
│   └ scala-3
└ build.sbt  
```

実際にこの構成をしてみたところ、 scala (バージョンなし）フォルダは利用しない方が良いでしょう。Scala 2.x と Scala 3.x の書き方が違いすぎるため、別フォルダに分けて各バージョンで適切な書き方をしておいた方が見通しが良いです。また、Scala 2.12 と Scala 2.13 でも、多少ソースコードの互換性が失われているので、これらも別々のフォルダに分けておいた方が良いです。

一方、Scala 2.12 と Scala 2.11 ではそこまで変化が少ないため、 Scala 2.11 -> Scala 2.12 でシンボリックリンクを作成しておいても良いでしょう。

## 異なる Scala バージョンで異なる設定を適用させる

build.sbt では Scala を書けるので、現在の適用バージョンを調べて match 式で分岐を書くことができます。例えば scalacOptions を Scala 2.x と Scala 3.x で書き換える場合：

```scala:build.sbt
scalacOptions ++= 
  CrossVersion.partialVersion(scalaVersion.value) match {
    case Some((3, _)) => // 3系
      Seq("-new-syntax", "-rewrite")
    case _ => // 2系
      Seq("-deprecation", "-unchecked", "-Xlint", "-Xfatal-warnings")
  }
```

# ネイティブライブラリを作る

GraalVM を使うことで、Jar ライブラリ以外にもネイティブで動くライブラリを作ることができます。ちなみに、[こちら](https://qiita.com/kijuky/items/533cb4389942d7143b97)で紹介したパーサー・コンビネーターを使ったライブラリくらいはネイティブ化できたりします（しました）。

```scala:project/plugins.sbt
addSbtPlugin("org.scalameta" % "sbt-native-image" % "0.3.1")
```

```scala:build.sbt
/* (さっきの記述から追記） */

lazy val cli = (project in file("cli"))
  .enablePlugins(NativeImagePlugin)
  .dependsOn(root) // メインのライブラリを使う、という設定
  .settings(
    scalaVersion := scala213,

    // ライブラリ設定。ゆるく CLI 書くなら、適当な cli ライブラリを導入すると便利。
    libraryDependencies ++= Seq(
      "com.github.losizm" %% "little-cli" % "0.8.0", // scala 2.13.x で使えるバージョン
      "commons-cli" % "commons-cli" % "1.5.0"
    ),

    // native イメージ生成用
    nativeImageOptions ++= Seq("-H:+AllowIncompleteClasspath"), // これがないとエラーになる
    nativeImageOutput := target.value / "native-image" / (ThisBuild / name).value // そのままだとプロジェクト名("cli")になっちゃうので、プロダクト名(`ThisBuild / name`)にする。
  )
```

```scala:cli/src/main/Main.scala
/* TODO: ほんとはいい感じの CLI を書く */
object Main {
  def main(args: Array[String]): Unit =
    println("Hello world")
}
```

cli プロジェクトに main 関数を持つクラスを用意しておきます。下記でネイティブイメージが作成できます。

```shell
$ sbt cli/nativeImage
  ...出力は省略...
$ cli/target/native-image/my-project
Hello world
```

簡単ですね。GitLab CI とかだと、 [hseeberger/scala-sbt](https://hub.docker.com/r/hseeberger/scala-sbt/):[graalvm-ce-21.3.0-java17_1.6.2_2.13.8](https://hub.docker.com/layers/hseeberger/scala-sbt/graalvm-ce-21.3.0-java17_1.6.2_2.13.8/images/sha256-ef5e9801096aeb9d9215ee9c242a4dece70fb93271802722187a649f796a36a2?context=explore) みたいな、GraalVM を内包する docker イメージをベースにビルドすると、gcc とかを用意する必要がないので便利です。

```yaml:.gitlab-ci.yml
native:
  image: hseeberger/scala-sbt:graalvm-ce-21.3.0-java17_1.6.2_2.13.8
  script:
    - sbt cli/nativeImage
  artifacts:
    paths:
      - cli/target/native-image
```

(Windows だとどうするんだろう...)

# まとめ

Scala プロジェクトから各種ライブラリを作る方法を紹介しました。 `crossScalaVersions` を与えることで、複数の Scala バージョンのビルド/パブリッシュを行うことができます。 `sbt-native-image` プラグインを使うことで、ネイティブライブラリを作成することができます。Scala バージョン間のバイナリ互換性がなくても、この sbt のサポートによってかなりハードルが低くなっています。また、ネイティブライブラリの作成ができることで、JVMに依存しない環境（クラウドのラムダだったり、非JVM系のWebサーバーだったり）でも、Scala の資産を活用することができます。

素晴らしいですね！みなさんも Scala 書きましょう！
