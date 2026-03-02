https://qiita.com/kijuky/items/af6c722ace6865c3eea2

---

「え、ScalaってJVM言語でしょ？ネイティブアプリは作れないよね？」と思ったそこのあなた！そう、あなたですよ！

確かにScalaはJVM言語です。ですが、JVM言語をネイティブ化する魔法のビルドツール、GraalVMを使うと、あら不思議！ついにScalaはJavaのランタイムがなくても実行できるようになります！

この記事ではScalaプロダクトをGraalVMでビルドする方法と、実際にビルドしたCLIを見ていくことにします。

# ScalaをGraalVMでビルドする

幸いなことに sbt-native-image という sbt plugin を使うことで、あなたは GraalVM の環境構築をすることなしに、GraalVM でビルドできてしまいます。便利ですね。

```project/plugins.sbt
addSbtPlugin("org.scalameta" % "sbt-native-image" % "0.3.2")
```

簡単なアプリなら、あとは `build.sbt` にこんな設定でいけます。

```build.sbt
lazy val root = (project in file("."))
  .settings(
    name := "sample",
    scalaVersion := "2.13.8", // 3系には対応していない。残念...
  )
  .enablePlugins(NativeImagePlugin)
  .settings(
    nativeImageVersion := "21.3.0", // 最新のバージョンを使う
    nativeImageOptions ++= Seq(
      "-H:+AllowIncompleteClasspath"
    )
  )
```

入門ということで HelloWorld を用意しておきましょう。

```src/main/scala/Main.scala
object Main extends App {
  println("Hello world");
}
```

これをビルド&実行してみましょう。

```shell
sbt nativeImage
target/native-image/sample
```

思ったより簡単でしょ？

## 例：LINE notify

では、これを用いて1つ例をみてみましょう。

https://github.com/kijuky/line-notify

このCLIツールは LINE notify を行います。

```
line-notify "hello world"
```

これくらいなら curl をそのまま叩けば良いかもしれませんが... Webリソースと連携して何かするCLIアプリなら難なくネイティブ化できます。 [Preference](https://docs.oracle.com/en/java/javase/19/core/preferences-api1.html), [Scopt](https://github.com/scopt/scopt), [sttp](https://github.com/softwaremill/sttp) などのライブラリも追加しても問題ありませんでした（ただし、Preferencesを有効にするためにGraalVMのバージョンを新しいものに指定しています）。

## 例：Slack notify

2つめ。似たようなことを Slack でもやってみましょう。[Slack は API が存在する](https://slack.dev/java-slack-sdk/guides/ja/)のでそれを利用してみます。

https://github.com/kijuky/slack-incoming-webhook

Slack API は内部で [Gson](https://github.com/google/gson) をつかうため、[リフレクションの設定が必要](https://docs.oracle.com/cd/F44923_01/enterprise/21/docs/reference-manual/native-image/Reflection/)になります。reflect-config.json を取得するために次のコードを実行します。

```shell
sbt
nativeImageRunAgent " hello world"
```

これで [`target/native-image-config/*.json` に、ビルドに必要なコンフィグファイル](https://github.com/kijuky/slack-incoming-webhook/tree/develop/target/native-image-configs)が出力されます。

# まとめ

簡単なScalaアプリなら、GraalVMを使ってネイティブアプリ化できました。実行時にJVMは不要ですし、実行速度も良好です。GraalVMはリフレクションが苦手ですが、コンパイルタイムで解決するScalaとは相性が良いです。皆さんもぜひScalaでネイティブアプリを試してみてください！
