https://qiita.com/kijuky/items/d33929c0d14f72fb3225

---

という挙動でバグらせたので、みなさんもご注意ください。

# Scala2.13だと

`else`を持たないif式、なかなか面白い挙動をします。

```scala
val a = if (true) "a"
```

これ、`a`は`Any`に推論されます。どういうことかというと`else ()`が省略されたような挙動になります[^1]。`String`と`Unit`の和を取って`Any`に推論される、という挙動です。`println`させると、実行時型に応じて出力されます。

```scala
val a: Any = if (true) "a"

println(a) // output: a
```

https://scastie.scala-lang.org/cNZonlgORBKr3yW1iNyugw

# Scala3だと

`else`を持たないif式は一律`Unit`として扱われます。

```scala
val a = if (true) a // Warning: A pure expression does nothing in statement position; you may be omitting necessary parentheses


println(a) // output: ()
```

https://scastie.scala-lang.org/LMcEiUGqToid9e3jhKXHUQ

警告文はかいつまんで言うと「`else ()`を忘れてんじゃないの？」と言ってます。実際、`else ()`を追加するとScala2.13のときと同じ挙動を示します。下記のような意見もあるので、else句のないif式は副作用の型すなわち`Unit`として扱われるのは一理あります。

https://x.com/gakuzzzz/status/1480197834721067010

# 問題は

このScala2と3の挙動の差がわかりにくいことに加え、条件によってはScala3の警告が発生しないことです。その例としてXMLリテラル内のif式があります。

```scala
val a = <a>{if (true) <b/>}</a> // nothing warnings!!!

println(a) // case Scala2 => <a><b/></a>
           // case Scala3 => <a></a>
```

つまり、既存コードに似たような物があると、警告なく異なるXMLが実行時に吐き出される、ということです。地獄ですね🔥。しかし今は令和の時代、そういうif式を見つけるscalafixがあるので皆さんは安心してマイグレーションできます。やったね！

https://xuwei-k.hatenablog.com/entry/2022/02/11/160802#:~:text=x%3A%20Int)%20%3D%3E%20x%20%7D-,NoElse,-https%3A//github.com

[^1]: より正確に言うと、`String`と`Unit`の和は`AnyVal`ですが、`if (true) "a"`は`Any`に、`if (true) "a" else ()`は`AnyVal`に推論されるようです。
