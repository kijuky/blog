https://qiita.com/kijuky/items/8c94702c6df97468a8bb

---

こんにちは。Scala 楽しいですよね。こんなに Scala が楽しいのは、楽しくするために発展していった歴史があった訳ですが、残念ながらその歴史の中には今では埋もれてしまったものたちもいます。この記事では sbt を使って、古い Scala バージョンのライブラリを遊んでみたいと思います。なんで sbt かはこちら。

https://qiita.com/kijuky/items/6e95db63689083de9e55

# 考古学班、出番です！

いくら古くても、GitHub にソースコードがあって、README.md に環境構築方法が書いてあるのなら、それに従うといいと思います。今回は Maven Repository に pom/jar があって、利用すべき Scala バージョンがわかる、と言う想定にします。

例えば、これとか。Maven Central には、掘っていくとたくさんのライブラリが置いてあって楽しいですね :skull:

https://mvnrepository.com/artifact/com.m3/m3util

ソースコード（GitHub）へのリンクもありましたが、閲覧権限がありませんでした。どうやって環境構築すればいいかわかりませんね。これを sbt を使って遊んでみます。

もし、対象とする Scala のバージョンと sbt のバージョンが揃っていれば、sbt 内部でそれを使うことができます。明示的なコンパイルが必要なくなるので、お試しするならまずはこれが便利です。ただし、全ての Scala バージョンが使えるわけではないので注意です（というか、この記事はこの表を書きたくて書いた）。

| Scala のバージョン | sbt のバージョン | 備考 |
|:-:|:-:|:-:|
| ~2.9 | NG | 先史時代 |
| 2.10 | 0.13.18 |   |
| 2.11 | NG |   |
| 2.12 | 1.5.5 |   |
| 2.13 | NG |   |

どこまで遡れるかな？と思って書いたけど、そもそも sbt が 0.13 以上じゃないと動かないっぽいから、 2.10, 2.12 以外なら素直にプロジェクト作った方がいいのかな...

気を取り直して、先のライブラリは 2.9, 2.10, 2.11 まで対応していたので、sbt 上で遊べます。ヤッタネ！

```project/build.properties
sbt.version=0.13.18
```

```project/plugins.sbt
libraryDependencies += "com.m3" %% "m3util" % "1.0.2"
```

この状態で IntelliJ を開くと sbt が依存しているライブラリが見れます（通常と違って jar ファイル名でソートされてるので探しづらいですが...）。型である程度想像できますが、Java にデコンパイルすれば気合い :muscle:  で読めます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/e123d54b-37de-48dd-6702-b134a0e13048.png)

文字列型から Option[Int] にしてますね。 parseInt の結果を Try ではなく Option で扱うためのライブラリでしょうか。さくっと build.sbt で動作確認してみましょう。

```build.sbt
lazy val t = taskKey[Unit]("")
t := Def.task {

  import com.m3.util._
  println(s"0 -> ${AsInt.unapply("0")}")
  println(s"+1 -> ${AsInt.unapply("+1")}")
  println(s"-1 -> ${AsInt.unapply("-1")}")
  println(s"1,000 -> ${AsInt.unapply("1,000")}")
  println(s"0.001 -> ${AsInt.unapply("0.001")}")
  println(s"${Int.MaxValue} -> ${AsInt.unapply(s"${Int.MaxValue}")}")
  println(s"${Long.MaxValue} -> ${AsInt.unapply(s"${Long.MaxValue}")}")
  println(s"a -> ${AsInt.unapply("a")}")
  println(s"0xa -> ${AsInt.unapply("0xa")}")
  println(s"z -> ${AsInt.unapply("z")}")

}.value
```

```
[IJ]> t
0 -> Some(0)
+1 -> Some(1)
-1 -> Some(-1)
1,000 -> None
0.001 -> None
2147483647 -> Some(2147483647)
9223372036854775807 -> None
a -> None
0xa -> None
z -> None
[success] Total time: 0 s, completed 2021/10/23 12:36:23
```

いい感じですね。16進数は扱えないようです。

# まとめ

こんな感じで、Scala 2.10, 2.12 であれば、 sbt から直接実行が確認できるため、昔のライブラリのちょっとした挙動を確認したい場合に便利かもしれません。ちなみに Java ライブラリであれば言語バージョンによる制限はないので、sbt 1.5.5 で遊べます。ただし、JVM 9 以上だとジグソーの影響で Runtime 依存を増やす必要があるかもしれません。それも面倒なら JVM のバージョンは 1.8 で試すと良いかもしれません。

インターネットの技術は日々進化していますが、温故知新とも言いますし、昔のライブラリで遊んでみるのもいかがでしょうか（頑張って綺麗にまとめてみた）。
