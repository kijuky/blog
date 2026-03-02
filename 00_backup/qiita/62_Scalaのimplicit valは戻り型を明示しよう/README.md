https://qiita.com/kijuky/items/6670041649cb1adcb34a

---

タイトル通りです。

# `implicit val`とは？

scalaには、単に変数宣言するだけではなく、「暗黙の値」を定義することができます。

```scala
implicit val a: String = "a"
```

このように宣言すると、これが宣言されたスコープにいたり、この宣言をインポートした時、aの値を明示せず、暗黙的に使うことができます。

```scala
def m()(implicit a: String): String = {
  s"implicit a = $a"
}

import a // ここで暗黙の値をインポート

m()(a) // 通常はメソッドを呼ぶためには引数をすべて設定する必要があるが、

m() // 暗黙の引数は、そのスコープにある暗黙の値を探そうとする。
```

# `implicit val`の型を省略すると警告が出る

暗黙の値は通常の変数宣言と同様に、型を省略することも可能です。

```scala
implicit val a = "a"
```

scala 2.13.11から、暗黙の値の型を省略すると警告が出るようになりました。

```
Implicit definition should have explicit type (inferred String)
```

https://github.com/scala/scala/pull/10083

暗黙の値を生成する時に、複雑な関数の戻り値だったりした場合、意図しない型に推論されて、暗黙の引数の解決ができなくなることがあるので、型を明示することは良いことだと思います。

# 今週あったこと

10/25にplay2.9が出ました。play2.9はscala xml2系に依存するようになったため、scoverageも最新バージョンを使えるようになって、scalaも最新バージョンを使えるようになりました。今まではscala 2.13.8でしたが、これでscala 2.13.12にアップデートできます。

対象のプロジェクトでは、play jsonのFormatを定義する際に、型定義を省略していました。

```scala
implicit val format = Json.format[ACaseClass]
```

これは警告対象です。なお、対象のプロジェクトでは`-Werror`でコンパイルしていたため、警告はコンパイルエラーとして報告されます。直すには下記のようにします。

```scala
implicit val format: Format[ACaseClass] = Json.format
```

いろんなjsonを扱うプロジェクトだったので、この書き換えが400ファイルくらいありました。しんどかったです。褒めて。

(quickfixも使えるらしいですが、quickfixだとFormatじゃなくてOFormatになりそうだったので、試してません...）

# まとめ

- implicit valには戻り型を明示しましょう。これにより暗黙の値の型が確定するので、暗黙の引数の解決が安定します。
- implicitに限らず、特に公開変数には可能な限り戻り型を明示しましょう。型が確定することで公開変数を扱いやすくなります。
- scala 2.13.xの警告はscala 3.xではエラーになることがあります。scala 2.13.xの警告はできる限り潰しておきましょう。
