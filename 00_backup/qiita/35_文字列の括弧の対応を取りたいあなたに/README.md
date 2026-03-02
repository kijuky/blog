https://qiita.com/kijuky/items/533cb4389942d7143b97

---

# 結論

Scala でパーサー・コンビネーターを書きましょう。

https://qiita.com/mattsu6/items/cb7f916242822b10b136

# 背景

よくある Play Framework を使った Web サイトで、「特定の条件下で要素を見せる・見せない」を制御したいことがありました。普通は（？） jQuery や Vue.js など、フロント側での表示制限を見ます。しかし、フロント側では取得できない情報に応じて表示制限をする場合、サーバーサイド側で HTML を加工して実現するようにしました。その際、HTML に下記のような要素があった場合、条件に応じて表示制限を行います。

```html
<template class="show" data-cond1="..." data-cond2="...">
  ...
</template>
```

`data-cond1` と `data-cond2` は最終的に論理和して真なら表示します。論理積はネストすることで表現できます。

```html
<template class="show" data-cond1="...">
  <template class="show" data-cond2="...">
    ...
  </template>
</template>
```

さて、これだと if-else みたいなのが簡単に表現できないのが辛いですね。A さんには表示させるけど、条件に当てはまらなかった B さんには別のコンテンツを見せたい、あると思います。これを表現するために data-not を用意しました。

```html
<template class="show" data-cond1="...">
  Aさんに表示
</template>
<template class="show" data-not="cond1(...)">
  Bさんに表示
</template>
```

data-not 内の書式は、 `data-cond1="..."` を `cond1(...)` と表記することで否定条件を取れるようにしてみました。

さて、これだと、条件が1つなら良いですが、（最初に挙げたような）条件が2つ以上の場合に記述することができません。そこで data-or を用意しました。全体の条件は論理和なので、最初の例を if-else を書く場合はこんなふうに書けます。

```html
<template class="show" data-or="cond1(...),cond2(...)">
  Aさんに表示
</template>
<template class="show" data-not="or(cond1(...),cond2(...))">
  Bさんに表示
</template>
```

論理和が取れるなら論理積も取りたいですよね。そこで data-and を用意しました。2番目の例は data-and を使うとネストせずに書けます。

```html
<template class="show" data-and="cond1(...),cond2(...)">
  ...
</template>
```

なるほどね。

# 実装

data-not 時に導入した `data-cond="..."` -> `cond(...)` の書き換えは結構便利です。というのも、 data-cond がたくさん定義されていたとしても、それらを `cond(...)` に変換して、最後に `or(...)` で囲えば、同じ字句解析処理に突っ込めます。ただし、問題が1つあって、この字句解析は括弧の対応を処理する必要があります。

文字列の解析ですぐに思いつくのは正規表現を書くことですが、残念ながらScala(Java)の正規表現では括弧の対応を取ることができません。括弧の対応を取るには自分で頑張って括弧の数を数え上げるか、パーサー・コンビネーターを利用する必要があるでしょう。

## パーサー・コンビネーター

パーサー・コンビネーターライブラリは、Scala のコップ本にも出てくるくらいに有名な Scala ライブラリの1つです。以前はコアライブラリになっていましたが、現在は外部ライブラリとして切り離されており、別途 sbt 等で依存関係を書く必要があります。

```scala:build.sbt
libraryDependencies += "org.scala-lang.modules" %% "scala-parser-combinators" % "2.1.1"
```

このライブラリは「パーサー」と「コンビネーター」の2つから構成されます。「パーサー」は与えられた文字列の字句解析を行います。「コンビネーター」は字句解析の結果から、独自クラスへの変換を行います。つまり、正規表現部分を「パーサー」が担っていて、その後の処理（ここでは、解析後の論理式を評価するためのインスタンスの作成）を「コンビネーター」が担っています。

この「パーサー」の特徴は [eBNF（拡張バッカスナウア記法）](https://ja.wikipedia.org/wiki/EBNF)と呼ばれる記法に似た記法で実装できることです。実際、今回の式を eBNF で書くことができれば、ほとんど実装できたと言っても過言ではありません。eBNF はプログラム言語の定義とかでチラッとみたことがあるかもしれません。難しい印象を持っている方もいるかもしれません（私がそうでした）が、正規表現を理解できるなら、それに加えて再帰の表現ができるようになった、と考えると理解がスムーズかもしれません。

### eBNF

それでは例題として、先程の論理式を eBNF で書いてみましょう。わざわざこの記法で整理する訳は、正規表現では表現できない、再帰を表現するためです。再帰の入り口として expr を定義します。

```bnf
expr = cond1 | cond2 | not | or | and
```

これは、式(expr)として `cond1`, `cond2`, `not`, `or`, `and` がある、という定義です。とりあえず　cond1 は条件として数字、 cond2 は条件として文字列が取れる、ということにしておきましょう。

```bnf
cond1 = "cond1" "(" #'\d+' ")"
cond2 = "cond2" "(" #'\w+' ")" 
```

`#'...'` は `...` に正規表現が取れるものと思ってください。 `\d+` なので数字の羅列、 `\w+` なので文字の羅列です。

さて、お待ちかねの not です。not は別の式を内部に持てます。

```bnf
not = "not" "(" expr ")"
```

not の定義に expr が出てきましたね。これが再帰の表現です。同様に or と and も定義してみましょう。

```bnf
or = "or" "(" expr { "," expr } ")"
and = "and" "(" expr { "," expr } ")"
```

`{ ... }` は、「0回以上の繰り返し」です。つまり、or や and はカンマ区切りで複数の式を内部に持てます。

これで eBNF での定義は終わりです。正規表現で書こうと思うと辛い感じですが、意外とあっさり書けて拍子抜けしていませんか？私はしました。以下に全体を再掲しますが、記述もとてもシンプルですね。この記述のままで動作確認できるサイトもあったりするので、実際に eBNF を書くときはそのようなサイトで動作確認しながら書くと良いでしょう。

```bnf
expr = cond1 | cond2 | not | or | and
cond1 = "cond1" "(" #'\d+' ")"
cond2 = "cond2" "(" #'\w+' ")" 
not = "not" "(" expr ")"
or = "or" "(" expr { "," expr } ")"
and = "and" "(" expr { "," expr } ")"
```

今回の記述は、こちらのサイトで動作確認できます。

https://mdkrajnak.github.io/ebnftest/

### パーサー

先程の eBNF を Scala のパーサー・コンビネーターライブラリで書き直した例が下記になります。

```scala:MyParser.sc
import scala.util.parsing.combinator.JavaTokenParsers

object MyParser extends JavaTokenParsers {
  def expr: Parser[Any] = cond1 | cond2 | not | or | and
  def cond1: Parser[Any] = "cond1" ~ "(" ~ """\d+""".r ~ ")"
  def cond2: Parser[Any] = "cond2" ~ "(" ~ """\w+""".r ~ ")"
  def not: Parser[Any] = "not" ~ "(" ~ expr ~ ")"
  def or: Parser[Any] = "or" ~ "(" ~ repsep(expr, ",") ~ ")"
  def and: Parser[Any] = "and" ~ "(" ~ repsep(expr, ",") ~ ")"
}
```

`~` がたくさん現れる以外には、ほとんど同じに見えませんか？アルコールが入った私にはほとんど同じに見えます。先ほどと違うのは or や and で `repsep` がありますが、これは repeat with separator の略で、「カンマ区切りで複数現れる要素」みたいなのを表します。 eBNF と違って expr が複数回現れなくてより良くなった感じしますね。しますね？

では、実際に実行してみましょう。サクッと [ammonite](https://ammonite.io) を使ってみます。

```shell
$ amm
Loading...
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import $ivy.`org.scala-lang.modules::scala-parser-combinators:2.1.1` 
import $ivy.$     
@ import $file.MyParser, MyParser._ 
Compiling ~/Desktop/MyParser.sc
import $file.$       , MyParser._
@ MyParser.MyParser.parseAll(MyParser.MyParser.expr, "or(and(cond1(123),cond2(abc)))") 
res2: MyParser.MyParser.ParseResult[Any] = Success(
  result = "or" ~ "(" ~ List("and" ~ "(" ~ List("cond1" ~ "(" ~ "123" ~ ")", "cond2" ~ "(" ~ "abc" ~ ")") ~ ")") ~ ")",
  next = CharSequenceReader()
)
```

それっぽく解釈されているように見えますね。ただ、これだけだとあまり嬉しくなくて、解釈された結果から論理式を実現するためにコンビネーターを使います。

### コンビネーター

コンビネーターはパーサーによって解析された結果から独自インスタンスを構築する仕組みです。今回は論理式を計算するためのインスタンスを作ってみましょう。

```scala:MyParser.sc
import scala.util.parsing.combinator.JavaTokenParsers

object MyParser extends JavaTokenParsers {
  def expr: Parser[Condition] = cond1 | cond2 | not | or | and
  def cond1: Parser[Condition] = "cond1" ~ "(" ~> """\d+""".r <~ ")" ^^ { Cond1Condition.apply }
  def cond2: Parser[Condition] = "cond2" ~ "(" ~> """\w+""".r <~ ")" ^^ { Cond2Condition.apply }
  def not: Parser[Condition] = "not" ~ "(" ~> expr <~ ")" ^^ { NotCondition.apply }
  def or: Parser[Condition] = "or" ~ "(" ~> repsep(expr, ",") <~ ")" ^^ { OrCondition.apply }
  def and: Parser[Condition] = "and" ~ "(" ~> repsep(expr, ",") <~ ")" ^^ { AndCondition.apply }
}

case class Context(i: Int, str: String)

trait Condition {
  def apply(context: Context): Boolean
}

case class Cond1Condition(value: String) extends Condition {
  def apply(context: Context): Boolean =
    value == context.i.toString
}

case class Cond2Condition(value: String) extends Condition {
  def apply(context: Context): Boolean =
    value == context.str
}

case class NotCondition(value: Condition) extends Condition {
  def apply(context: Context): Boolean =
    !value.apply(context)
}

case class OrCondition(values: Seq[Condition]) extends Condition {
  def apply(context: Context): Boolean =
    values.exists(_.apply(context))
}

case class AndCondition(values: Seq[Condition]) extends Condition {
  def apply(context: Context): Boolean =
    values.forall(_.apply(context))
}
```

一気に実装を書いてみました。先ほどとの違いをみてみましょう。

まずは `Parser[Any]` が `Parser[Condition]` になっているのがわかると思います。これはパース結果として Condition インスタンスを返すことを意味します。そのために各パーサーの結果が Condition になるように ` ^^ { 〇〇Condition.apply }` が追加されています。

```scala
  def cond1: Parser[Condition] = "cond1" ~ "(" ~> """\d+""".r <~ ")" ^^ { Cond1Condition.apply }
```

これは、解析した結果を引数に Cond1Condition ケースクラスのコンストラクタ(apply メソッド)を呼び出しています。 `^^` の左側がパース部分、左側がコンビネーター部分です。

パース部分もちょっとだけ変わっていて `~ """\d+""".r ~` が `~> """\d+""".r <~` となっています。これは `^^` に渡される結果が `~>` と `<~` の間で囲われた部分だけ渡すことを表します。通常は、このように可変部分だけを囲んであげると便利です。正規表現のパース結果は `Parse[String]` となるため、 `^^` には String の引数が渡されます。そのため `Cond1Condition.apply` は String の引数を取るようにしています。ちょっとしたパズルですね。

今回は例として、Context に渡された文字列と同じかどうか、ということにしてみましょう。

```scala
case class Cond1Condition(value: String) extends Condition {
  def apply(context: Context): Boolean =
    value == context.i.toString
}
```

その他の関数もそれっぽく実装しましょう。not は単に否定をとれば良いです。or と and は便利な関数 exists, forall が Seq に生えているので、それを使いましょう。

それでは、先ほどと同様に ammonite で実行してみましょう。

```shell
$ amm
Loading...
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import $ivy.`org.scala-lang.modules::scala-parser-combinators:2.1.1` 
import $ivy.$     
@ import $file.MyParser, MyParser._ 
Compiling ~/Desktop/MyParser.sc
import $file.$       , MyParser._
@ MyParser.MyParser.parseAll(MyParser.MyParser.expr, "or(and(cond1(123),cond2(abc)))").get.apply(Context(123, "abc")) 
res2: Boolean = true
```

確かに

- `cond1(123)` は context で与えた数字と一致するので真
- `cond2(abc)` は context で与えた文字列と一致するので真
- `and(cond1(123), cond2(abc))` は、両方とも真なので真
- `or(and(cond1(123), cond2(abc)))` は、中身が真なので真

なので、計算できてそうですね。

## Jsoup

ここからはおまけですが、例として出した template タグの表出制御はこんな感じでできます。template タグは何も処理なければ非表示になるので、この処理が走らなくても影響を最小限にできます。条件を `data-*` の形でまとめたのは dataset として取得するためです。並列に書かれた `data-*` の論理和を取るのは「`template` タグがデフォルトで非表示」と「`data-*` が1つもないときは偽として非表示」が一致するためです。

```scala
import org.jsoup.Jsoup
import org.jsoup.nodes.Element
import scala.jdk.CollectionConverters._

object TemplateProcessor {
  def process(context: Context): String = {
    val document = Jsoup.parseBodyFragment(body)
    document.select("template.show").asScala.foreach { element =>
      if (isShow(element, context)) {
        // template タグ内の要素を template タグの上に展開する。
        element.childNodes.asScala.foreach(element.before)
      }
      // template タグ自身を削除する。
      element.remove()
    }
    document.body.html
  }

  def isShow(element: Element, context: Context): Boolean = {
    val conditions = element.dataset.asScala
      .map { case (key, value) => s"$key($value)" } // data-cond="..." を cond(...) に書き換える。
      .mkString(",") // or の引数にするために各式をカンマ区切りにする。
    val result = MyParser
      .parse(s"or($conditions)") // 並列に書かれた data-* は論理和を取る。
      .map(_.apply(context))
      .getOrElse(false)
  }
}
```

# まとめ

括弧の対応が取りたくなったら Scala のパーサー・コンビネーターライブラリを使いましょう。eBNF と似た記述で実装できるため、正規表現と比べて視認性も良いです。
