https://kijuky.hatenablog.com/entry/2021/06/08/223540

---

[rpscala 勉強会](https://www.youtube.com/channel/UCeNa-erztsFtVTT8rmZw-hQ) に出てみた。自分は監視枠（Youtube コメント枠）

[質問箱](https://peing.net/ja/rpscala1?p=auto&utm_source=twitter&utm_medium=timeline&utm_campaign=auto_recruitment)に答えたあと、モブプロをしていた。

質問箱1つ目　例外、Either、その他の使い分けは？

https://peing.net/ja/q/d34a1acb-55c4-4abd-9805-ae9c8268f0ae

質問箱2つ目　Javaとの優位性は？　途中で Kotlin の比較も追加された。

https://peing.net/ja/q/18181f38-7084-4398-9147-58a6b4b77a89


モブプロでは Scala 3 のマクロについて、解説ページを読んだあとに、実際に手を動かしていた。

https://softwaremill.com/scala-3-macros-tips-and-tricks/

`inline` の挙動についてコンパイルできるかどうかを確認していた。

- `inline` 関数の呼び出しに使われる変数は `inline` じゃないとだめ
- `inline` の再帰関数も動く
    - 停止しないような再帰関数(32回よりも多く展開するような関数)を書くと、コンパイルエラーになる。
    - `32` はオプションで変更可能

```scala
inline def power(v: Int, p: Int): Int = {
  if p == 0 then v // このへんちょっと Scala 3 で書いてみた♪　コンパイル通るよ
  else v * power(v, p - 1)
}

@main def a(): Unit = {
  println(power(2, 32))  // 33 だとエラー
}
```

- `inline` は事実上の `final`(オーバーライド不可)になる
    - コンパイル後に消えてしまうので。
- `inline` を使えば、メソッドの引数の型を動的に変えることもできる。
- Scala2 で macro でやってたことは `inline` でできる事があるので、まずは `inline` で実装してみると良さそう。

型クラスの話

- 型クラスは Haskell を意識している（らしい。自分は Haskell 書いたことない...）
    - inline と using Mirror を使うと、マクロを書かなくても型クラスの導出ができる...らしい...
- インデントベースはコピペが難しい
    - これはIDEの支援を待つ感じなんだろうか...

# 感想

- 例外、Either の使い分けはコンテキストに依存しそうな気がした。
    - Javaベースだと例外がしっくり来るし、Scalaベースだと Either がしっくりきそう
- Java/Scala のメリデメは色々あるが、個人的にはバージョン互換性大事だと思った。
    - オープニングでも Scala 2.11 -> 2.12 に難儀している話があった。
    - 自分も Scala と Kotlin のバージョンアップをやったことがあるが、圧倒的に Kotlin の方が楽だった。
- inline みたいな機能は、むしろ今まで Scala 2 ではマクロを使わないといけなかったのが大変だったんだなぁと思った
    - Scala 2のマクロは使ったことない
- 型クラスはついていけなかった...

# 参考

`inline` に関する公式ドキュメント

https://docs.scala-lang.org/scala3/guides/macros/inline.html

型クラスに関する公式ドキュメント

https://dotty.epfl.ch/docs/reference/contextual/derivation.html

