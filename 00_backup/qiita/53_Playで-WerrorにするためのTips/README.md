https://qiita.com/kijuky/items/a7880c05a6157dabedd8

---

`-Werror`とは、Scalaコンパイラの警告をエラーと見なす[オプション](https://docs.scala-lang.org/overviews/compiler-options/index.html)です。Scalaコンパイラの警告はランタイムエラーの予兆ですので、できる限り潰しておきましょう。

https://eed3si9n.com/ja/stricter-scala-with-xlint-xfatal-warnings-and-scalafix/

さて、今回は[Play](https://www.playframework.com/)プロジェクトで`-Werror`を有効にするための、便利なScalaコンパイラオプションを紹介します。

# 便利なScalaコンパイラオプション

## `-Wconf:any&src=target/.*:s`

いきなり呪文みたいなオプションですが、これはtargetフォルダにあるソースコードの警告を抑制するオプションです。Playはroutesや[twirl](https://github.com/playframework/twirl)など、DSLからScalaコードを自動生成しますが、そのいくつかは警告が発生してしまいます（特に多いのが`unused import`）。自動生成コードの警告はアプリケーション側では何とも対処できないため、警告の発生を抑制させてしまいましょう。

https://twitter.com/takezoen/status/1378717430902480899

## `-Wmacros:after`

特に[macwire](https://github.com/softwaremill/macwire)を使っているときに多いのですが、マクロ適用時に警告が出ることがあります。この設定で、マクロ展開時の警告を抑制します。

# 参考

https://docs.scala-lang.org/overviews/compiler-options/index.html
