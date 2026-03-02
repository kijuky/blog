https://qiita.com/kijuky/items/726f4e36ba3561b09a23

---

皆さんごきげんよう。最近は社内のプロジェクトをScala3/Play3にする職人と化しています。そんな作業の中で、既存プロジェクトがScala3対応していないライブラリを使っていたりするとなかなか移行の判断が難しくなったりします。この記事では、2024年9月時点で、既存Play2.9/Scala2.13プロジェクトをPlay3.0/Scala3.3に移行するにあたり、ライブラリの移行の指針について紹介します。あくまで一例であり、「うまくいうかもしれないよ」程度でご覧ください。

# 「Scala3対応していないライブラリ」とは

ここでは、**そのライブラリの2.13バージョンを利用できないもの**を指します。Scala3は後方互換性があり、Scala2.13向けのライブラリを使えます。便利ですね。そのため、たとえScala3向けのバイナリが提供されていなくても、

```scala
"groupId" %% "artifactId" % "version"
```

これを、

```scala
"groupId" % "artifactId_2.13" % "version"
```

にすると使える可能性があります。

# Scala3対応していないライブラリ

以下では、前述した対応では対処できないライブラリについて代替案を提示します。

## shapeless

https://github.com/milessabin/shapeless

shapelessのSizedやHListは、Scala3のタプルが代替案になります。公式ドキュメントをみれば、書き換えも容易でしょう。困ったならCopilotに聞くと、良い感じにコード生成してくれます。

https://www.scala-lang.org/2021/02/26/tuples-bring-generic-programming-to-scala-3.html

## ficus

https://github.com/iheartradio/ficus

hocon設定ファイルを任意のケースクラスにバインドするライブラリです。Playを使っている場合、Play2.6でConfigLoaderが提供されたため、それを使うことができます。

https://www.playframework.com/documentation/latest/ScalaConfig#ConfigLoader

## mockito-scala

https://github.com/mockito/mockito-scala

mockitoを自然なDSLで書けるようにするライブラリです。Java版mockitoと比べてあまりにも自然に読み書きできるのでmockito DSLであることを忘れてしまうくらい素晴らしい出来ですがScala3対応していません。私はscalatestplus.mockitoを使うことにしました。

https://github.com/scalatest/scalatestplus-mockito

## play-json4s-jackson

https://github.com/tototoshi/play-json4s

playでjson4sを使えるようにするライブラリです。そもそものplayのバージョンアップに追随できていないので、代替を探す必要があります。候補としてはjson4sを直接使ったり、そのほかのライブラリに移行することになります。ちなみに私はplay-jsonを使いました。

## play2-pager

https://github.com/gakuzzzz/play2-pager

playでページャーを自動的に作成してくれる便利ライブラリです。これもplayのバージョンに追随できていないので代替を探す必要があります。私はフォークして自前でパブリッシュして対応しました。

## metrics-play

https://github.com/kenshoo/metrics-play

playのメトリクスを取得できるフィルターを提供するライブラリです。オンプレだと便利ですが、クラウドだと各種メトリクスはそのクラウドベンダーがいい感じのものを提供してくれるので、不要そう。

## play2-memcached

https://github.com/mumoshu/play2-memcached

playでmemcachedを使えるようにするライブラリです。同様に自前でパブリッシュして対応しました。（本当はコントリビュートしたかったけど、現状のプロジェクトだと自分の技術力では無理そうだった...）

## sbt-scalate-precompiler

https://github.com/scalate/sbt-scalate-precompiler

scalateで使えるテンプレートファイルを事前にscalaコードにコンパイルすることで、文字列解析のコストをゼロにします。現在は自前でパブリッシュしたものを使っていますが、プルリクの進行度合いによっては本家対応されるかもしれません。

## 社内ライブラリ

こういうのが一番辛かったりします。そもそもScala3が発表されて3年も経っているので、今の時点で対応していないなら、もう自分が当事者になるしかありません。幸いsbtはscalaバージョンのクロスビルドに対応しているので、丁寧に設定ファイルをいじればscala2/scala3クロスビルド対応することが可能です。

https://www.scala-sbt.org/1.x/docs/Cross-Build.html

ただし、scala3ビルドする場合はsbt1.5以上が必要になります。そもそもsbtのバージョンが古すぎる場合はそこから対応する必要があります。

# まとめ

Scala3対応していないライブラリの紹介と、その代替案を提示しました。基本戦略としては、OSSならコントリビュート、社内ライブラリなら社内政治によってScala3ライブラリをパブリッシュするように動く、ダメなら自前でホストして使えるようにする、それでもダメなら代替案を探す、という感じになります。ただ、ScalaやPlayもバージョンアップによって使える機能が増えたため、単に代替ライブラリを探すのではなくScalaやPlayの機能を改めて確認するのも重要になってくるかもしれません。また、ここに挙げたのはあくまで現時点での解決策の一例であり、異なる解決策だったり、時間が経ったことでScala3対応が進むことで異なる解決策がベストになる可能性があるため、最新状況はご自身でご確認いただく必要があることにご注意ください。

以上です。対応した暁には、皆さんも良きScala3ライフをご堪能ください。
