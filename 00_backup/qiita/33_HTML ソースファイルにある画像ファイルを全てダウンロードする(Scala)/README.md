https://qiita.com/kijuky/items/cac40eb3ba1d4dcd9df6

---

例えば HTML コンテンツの画像ファイルが [Zope](https://ja.wikipedia.org/wiki/Zope) で管理されていて、それらのファイルを一括で取得したいのだけど [zexp](https://whatext.com/ja/zexp) というファイルに固められてにっちもさっちもいかなくなっている皆さん、こんにちわ。そんないつの時代の話をしているんだい？という状況になくても、例えば正規表現で表せられるくらいの URL を一括でダウンロードしたいとかいうことありませんか？私はあります。こんなときも sbt を使って「面倒なことは Scala にやらせ」ましょう。なんで sbt かはこちら

https://qiita.com/kijuky/items/6e95db63689083de9e55

# 前提条件

- あなたは html 記事コンテンツのソースコード(html.txtファイル)を持っています。
- その html ファイルにリンクされている画像を全部ダウンロードしたいものとします。
- 保存場所はパスを維持するものとします。
    - http://example.com/a/b/c/image.png があるとすれば、 ./a/b/c/image.png に保存することにします。

# 記事コンテンツを読み込む

Java でローカルファイルを読み込む方法は滅多にやらないので忘れがち。とりあえずささっとやりたい場合は nio が便利でした。

```scala
import java.nio.file._
val file1 = Paths.get(filename)
val text = Files.readString(file1)
println(text) // 標準出力に表示される
```

ちなみに、今回私が扱おうとしていたファイルは複数の記事コンテンツをまとめたものであり、660 MB ほどあって、これを sbt で読み込ませたところ OOM が発生しました。これくらいの容量のファイルを読み込ませる場合は適当に使用メモリを増やしておきましょう。

```bash:.envrc
export SBT_OPTS="-Xms1g -Xmx8g -XX:ReservedCodeCacheSize=256m -XX:MaxMetaspaceSize=512m"
```

# 画像リンクのリスト

先程のテキストに正規表現を使って画像リンクのリストを作ります。正規表現がややこしいことを除けば、特に説明はいらないでしょう。

```scala
val results = """['"]([^'"]+?//example.com/.+?)['"]""".r // 正規表現は適当
  .findAllMatchIn(text)
  .map(_.group(1))
  .toSeq
  .distinct
```

特筆すべきは、正規表現結果は Iterator で返ってきて、それに toSeq をすると Stream(LazyList) で返ってくることでしょうか（sbt 内はまだ scala 2.12 なので Stream 型）。ただ、最後に distinct しているので、どれだけ遅延評価が役に立っているかは正直よく分かってません...

# 画像ファイルのダウンロード

ファイルのダウンロードってめんどくさそう... と思っていたのですが、高度に抽象化された Java では `copy` という操作になるようです。いつの間にそんなに便利になったの？

```scala
import java.io._
import java.nio.file._
import scala.util.Try
results
  .map(_.replace("¶", "").trim) // （改行など）余計な文字がくっついている場合は適当に変換をかける
  .filter(r => Seq(".jpg", ".JPG", ".jpeg", ".JPEG", ".png", ".PNG", ".gif", ".GIF", ".svg", ".SVG").exists(r.endsWith)) // 画像ファイルだけ取ってくる
  .foreach { result =>
    val url1 = url(result)
    val file1 = file(s".${url1.getFile}")
    file1.getParentFile.mkdirs() // 保存先フォルダの作成。作っておかないと Files.copy で失敗する。

    Try {
      io.Using.resource((_: URL).openStream())(url1) { input => // 慣れるまでは sbt.io.Using は書きづらい気がする...
        Files.copy(input, file1.toPath, StandardCopyOption.REPLACE_EXISTING) // ダウンロード処理（1行だけ！！！）
      }
    } recover {
      case _: FileNotFoundException => // リンク切れなどでファイルがなかった場合。 results.foreach の中にいるので、例外は握りつぶして後続のファイルダウンロードを続ける。
        System.err.println(s"$result not found.")
    }
  }
```

# まとめ

最後に全部まとめて build.sbt にしたのがこちら。

```bash:.envrc
export SBT_OPTS="-Xms1g -Xmx8g -XX:ReservedCodeCacheSize=256m -XX:MaxMetaspaceSize=512m"
```

```build.sbt
import scala.util.Try

lazy val download = taskKey[Unit]("")
download := Def.task {
  fetch("html.txt")
}.value

def fetch(filename: String): Unit = {
  import java.io._
  import java.nio.file._

  val file1 = Paths.get(filename)
  val text = Files.readString(file1)
  val results = s"""['"]([^'"]+?//example.com/.+?)['"]""".r
    .findAllMatchIn(text)
    .map(_.group(1))
    .toSeq
    .distinct
  results
    .map(_.replace("¶", "").trim)
    .filter(r => Seq(".jpg", ".JPG", ".jpeg", ".JPEG", ".png", ".PNG", ".gif", ".GIF", ".svg", ".SVG").exists(r.endsWith))
    .foreach { result =>
      val url1 = url(result)
      val file1 = file(s".${url1.getFile}")
      file1.getParentFile.mkdirs()

      Try {
        io.Using.resource((_: URL).openStream())(url1) { input =>
          Files.copy(input, file1.toPath, StandardCopyOption.REPLACE_EXISTING)
        }
      } recover {
        case _: FileNotFoundException =>
          System.err.println(s"$result not found.")
      }
    }
}
```

build.sbt 内では file とか url とかを変数名にすると sbt.file とか sbt.url とかとバッティングするので、 file1 とか url1 とかにしておくと良さそう。

# 余談

全て終わった後に気づいたのですが、多分欲しかったのは Irvine でした。

https://forest.watch.impress.co.jp/library/software/irvine/
