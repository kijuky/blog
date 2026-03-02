https://qiita.com/kijuky/items/64ea7e1c95ede6799a24

---

Scala 3 出ましたね！乗るしかない、このビッグウェーブに！

でも、Play Framework の Scala 3 対応まだだしなぁ、、、と思っているそこのあなた！何も Scala で動く Web フレームワークは Play だけではありません。最近流行りの SpringBoot を使って、 Scala 3 を使った Web アプリケーションを見てみましょう。

# 結論

最近流行りの　SpringBoot の環境構築は Gradle + Kotlin + Vue.js で行われていますが、これを Scala ナイズしましょう。その名も Sbt + Scala + **Scala.js** です。

https://github.com/kijuky/springboot-scala.g8

```shell
sbt new kijuky/springboot-scala.g8
```

おーっと、これは Scala 2.x ですね。それでは本題の Scala 3 をやってみましょう

```shell
sbt new kijuky/springboot-scala.g8 --branch scala3
```

[README.md](https://github.com/kijuky/springboot-scala.g8/blob/main/src/main/g8/README.md) に書いてあるとおり、下記を実行すると `It works` が見られます。

```shell
docker compose up -d
sbt server/run
```

```shell
open http://localhost:8080/
```

この環境構築さえできてしまえば、あとは SpringBoot を Scala 3 で書くことができます。

# 経緯

## Gradle プロジェクトを Sbt プロジェクトにした

そもそも最初は Gradle に Scala サポートがあったので、そちらで環境構築していました。ただ、フロントエンドに何使おうかなぁ？と考えたところ、わざわざ Scala 使うんだから、Scala と親和性あるやつがいいよね？と思い、Scala.js を選んでみました。Scala.js も Gradle で環境構築できなくはないですが、Sbt ベースの場合、JavaScript の配置に悩まなくて良くなるし、Sbt だと Scala 3 の環境構築が楽なので、Gradle にはさよならしてもらいました。

参考にした記事

- [Gradle + SpringBoot + Scalaで開発始める始める方法 - NBM2](http://kazuhito-m.github.io/tech/2016/11/12/gradle-springboot-scala)
- [SpringBootプロジェクトをScala + sbtで構築する - 冥冥乃志](http://mao-instantlife.hatenablog.com/entry/2015/09/07/SpringBoot%E3%83%97%E3%83%AD%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92Scala_%2B_sbt%E3%81%A7%E6%A7%8B%E7%AF%89%E3%81%99%E3%82%8B)
- [bjoernjacobs/spring-boot-scala: An example project ... - GitHub](https://github.com/bjoernjacobs/spring-boot-scala)

すでに SpringBoot + Kotlin やってる人から見れば、書きっぷりはあんまり変わんないですね。自分はテンプレートエンジンにロジックの少ない mustache が好きなのでそれを入れました。case classからutil.Mapにするおまじないを入れてあげると、なかなか便利にレンダリングしてくれます。

```scala
case class ViewModel(
  value: String = "It works!"
) {
  def toMap = {
    import scala.jdk.CollectionConverters._
    this.getClass.getDeclaredFields.map(_.getName).zip(this.productIterator.toList).toMap.asJava
  }
}
```

```scala
@Controller
class IndexController {
  @RequestMapping(path = Array("/"), method = Array(RequestMethod.GET))
  def sample(): ModelAndView =
    new ModelAndView("index").addAllObjects(ViewModel().toMap)
}
```

```mustache:index.mustache
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>title</title>
</head>
<body>
{{ value }}
</body>
</html>
```

## Scala.js の導入

次に、フロント用に Scala.js を導入しました。軽く説明すると Scala.js はコンパイルすると1つの main.js になるので、SPA に向いてます。Play を使った Scala.js のマルチプロジェクトの例はよくあるのですが、 SpringBoot でも環境構築はできます。SpringBoot の場合は Akka の設定が役に立ちました。

- [Scala.jsことはじめ - Qiita](https://qiita.com/Takashi_Kasuya/items/77385aa1fc8080778368)
- [vmunier/akka-http-scalajs.g8: Akka HTTP with Scala.js - GitHub](https://github.com/vmunier/akka-http-scalajs.g8)

ほぼほぼ Play と一緒ですが、圧縮に関しては SpringBoot 側に設定があるので、Sbt ではなくそちらに設定します。Scala が書ける JavaScript なんて、トランスパイル結果は膨大になりますから、圧縮は必須ですね。圧縮によって大体サイズが 20% 位になります。

```yaml:application.yml
server:
  compression:
    enabled: true
```

## そして Scala 3 へ

ここまでくれば、あとは Sbt の scalaVersion を 3.0.0 にするだけです。Play の依存関係のつらみを知っている人にとってはあっけなく感じることでしょう...（インデント構文は趣味です）

- [Scala 3 対応の差分](https://github.com/kijuky/springboot-scala.g8/compare/main...scala3)

さて、ここで残念なお知らせです。Scala.js は Scala 3 に対応しているものの、 Scala.js の各種ライブラリが Scala 3 に対応していません。Scala.js では、基本的な DOM 操作も [scalajs-dom](https://index.scala-lang.org/scala-js/scala-js-dom/scalajs-dom/1.1.0) のようなライブラリを利用するため、これらのライブラリが使えないとなるとなかなかつらいです...

### 追記

Scala 3 は Scala 2.13 のバイナリが読めるので、 `%%%` ではなく `%` で書けば scalajs-dom も利用できました。やったね！

```scala
libraryDependencies += "org.scala-js" % "scalajs-dom_sjs1_2.13" % "1.1.0"
```
