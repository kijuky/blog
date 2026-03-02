https://qiita.com/kijuky/items/5b172cb285586feea7a3

---

私が初めて Scala を触ったのが [Eclipse](https://www.eclipse.org/downloads/) + [ScalaIDE](http://scala-ide.org/) なのですが、その時からソースコード自動整形ツールが導入されており、複雑な for 文やメソッドチェーンも「なるほどこういう風に改行やインデントを工夫すると見やすくなるのかー」と感心したものです。最近では業務やOSSなどで複数人で開発していると、ソースコードに対するスタイルの考え方の不一致も増えてきて、レビュー指摘が荒れたりもしますが、そんなときでも自動整形ツールがあれば、そのツールの出力に従うというルールのもとで開発を円滑に進めることができます。そんな、最近では必ず設定されるソースコード自動整形ツールについてちょっと調査したのでその備忘録＆ポエムとして残しておきます（Scalaが対象です）。

# Scalariform

https://github.com/scala-ide/scalariform

ScalaIDE のフォーマッター機能である Scalariform は、昔から使われていたので、ある程度歴史がある Scala プロダクトのフォーマッターでは使われていたりします。ScalaIDE の他、[sbt プラグイン](https://github.com/sbt/sbt-scalariform)もあるため、Eclipse ベースではないプロジェクトでも利用可能です。

ただ、残念ながら[更新が途絶えていて（現時点で2019年が最新版）](https://github.com/scala-ide/scalariform/releases/tag/0.2.10)、Scala3 も出た昨今、ちょっとサポートに難があるかもしれません。

IntelliJ IDEA でサポートされないのも辛いです（[検索するとプラグインがヒットする](https://plugins.jetbrains.com/plugin/7480-scalariform)が、最新版のIntelliJ IDEAにはインストールできない）。

# Scalafmt

https://github.com/scalameta/scalafmt

もう一つの Scala フォーマッターである Scalafmt は [IntelliJ IDEA に標準対応](https://pleiades.io/help/idea/work-with-scala-formatter.html)しています。更新も活発で、[Scala3 にも対応しています](https://scalameta.org/scalafmt/docs/configuration.html#scala-3)。とりあえずフォーマッター入れたいのであれば、Scalafmt にしておくのが今のところベターなんですかね？

## 導入方法

せっかくなので既存 sbt プロジェクトへの導入方法です。といっても、やることは少なくて plugins.sbt への追加と `.scalafmt.config` を追加するだけです。

```plugins.sbt
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.4.0")
```

```config:.scalafmt.config
version = 3.0.0-RC6
preset = IntelliJ
```

これで `sbt scalafmtAll` とすれば、配下にある scala ファイル全てをフォーマットしてくれます。

```
$ sbt scalafmtAll
```

らくしょーですね！


## .scalafmt.config

[色々と設定があります](https://scalameta.org/scalafmt/docs/configuration.html)が、個人的にはバージョンの指定と[presetの指定](https://scalameta.org/scalafmt/docs/configuration.html#presets)だけにしたほうがいいかなーと思っています。

### version

このバージョンは scalafmt-core のバージョンです。省略すると latest が設定されます。省略した場合の難点として、 scalafmt-core のバージョンが上がったときに既存のフォーマットに変更がある可能性があります。また、パッチバージョンアップであっても、フォーマットに変更がないことが保証されません。そのため、この規則を省略していると、いつの間にかフォーマット規則が変更されて自分が変更していないコードのフォーマット結果がコミットに含まれるとかいう地獄が形成されかねません。

### preset

いくつかの設定をまとめたものです。下記があるようです。

- default
- IntelliJ
- defaultWithAlign
- Scala.js

よほど凝ったことをしたいということがない場合は default か IntelliJ にしておくと良いでしょう。

## 備考

フォーマットされていないコードを scalafmt でフォーマットした場合、稀にフォーマットされないことがあります。その場合は2~3回 `sbt scalafmtAll` してみて、出力結果が変わらないことを確認してからコミットしましょう。

# まとめ

- Scalariform は古くなってしまったので、今からフォーマッターを導入する場合は Scalafmt を使いましょう。
- ところで、既存のプロジェクトに Scalafmt をいれて `sbt scalafmtAll` をするとめっちゃ差分でるんですが、こういったフォーマッターの入れ替えや適用って皆さんどうしていますか？他の進行中のブランチとの兼ね合いもあるし、上手い導入の仕方を知っている人がいたら教えて欲しいです :pray:
