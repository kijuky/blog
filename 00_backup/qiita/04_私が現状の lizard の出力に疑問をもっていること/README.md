https://qiita.com/kijuky/items/fa9f9eedfda9ea716106

---

# この記事について
- [循環複雑度(CCN)](https://ja.wikipedia.org/wiki/%E5%BE%AA%E7%92%B0%E7%9A%84%E8%A4%87%E9%9B%91%E5%BA%A6)と、それを出力するツール [lizard](https://github.com/terryyin/lizard) について調べたことをまとめています。
- lizard は [fastlane](https://github.com/liaogz82/fastlane-plugin-lizard) や [sonar](https://github.com/Backelite/sonar-swift/blob/develop/docs/sonarqube-fastlane.md) のプラグインになっています。[lizard の出力を元にした記事](https://speakerdeck.com/imaizume/tips-of-swift-programming-to-reduce-code-complexity)もあります。私の疑問の解決の仕方によっては、同プラグインを利用して静的解析している人たちや、同記事の内容に少なからず影響を与えることになると感じています。そのため、lizard にプルリクを投げる前に議論の場が欲しいと思いました。
- コメントお待ちしております。

# はじめに
業務で既存コード(言語は Swift)のテストカバレッジを上げるため、既存コードの CCN を [lizard](https://github.com/terryyin/lizard) で測定し、テストコード数の参考にしていました。そこに下記のようなコードがあり、lizard から「CCN が高い」と警告されました。

```swift:code1.swift
func f0() {
  something(for: .what)
  something(for: .what)
  ... 15 個くらい続く
  something(for: .what)
}
```

このコードを次のように変更すると lizard が出力する CCN は 1 になりました。

```swift:code2.swift
func f0() {
  something(what: .what)
  something(what: .what)
  ... 15 個くらい続く
  something(what: .what)
}
```
[Swift の言語仕様](https://docs.swift.org/swift-book/LanguageGuide/Functions.html#//apple_ref/doc/uid/TP40014097-CH10-ID166)的なことを説明すると、`for:` や `what:` は「引数ラベル(argument label)」と言って、上記例のように `for` や `if` などのキーワードであっても指定できます。

この実行結果を受けて、「lizard はソースコードのコンテキストを無視して `for` や `if` が現れた回数をカウントしているだけなんじゃないか？」という疑問が湧きました。

# lizard のコードリーティング

lizard は GitHub で公開されている OSS でしたので、ソースコードを確認できました。そして実際に[「`for` や `if` が現れた回数をカウントしている」箇所](https://github.com/terryyin/lizard/blob/master/lizard.py#L509)がこちらになります。

```python:lizard.py
def condition_counter(tokens, reader):
    conditions = reader.conditions
    for token in tokens:
        if token in conditions:
            reader.context.add_condition()
        yield token
```

`tokens` はソースコードを単語分解したリストが入り、`conditions` は[次のように](https://github.com/terryyin/lizard/blob/master/lizard_languages/code_reader.py#L85)定義されています。

```python:code_reader.py
class CodeReader(object):
    # ...省略
    _conditions = set(['if', 'for', 'while', '&&', '||', '?', 'catch',
                       'case'])
    # ...省略
    def __init__(self, context):
        # ...省略
        self.conditions = copy(self._conditions)
```

これで確かに「ソースコードのコンテキストを無視して `for` や `if` が現れた回数をカウントしている」ことがわかりました。

## lizard のポリシーを知る

言語処理系に詳しい人たちからすると、こーいう計測は抽象構文木を構築して分岐を探したほうが良いという感想を持つかもしれません。しかし lizard はあえてそれをしていません。それは [README](https://github.com/terryyin/lizard/blob/master/README.rst) で言及されています。

> This tool actually calculates how complex the code 'looks' rather than how complex the code really 'is'. People will need this tool because it's often very hard to get all the included folders and files right when they are complicated. But we don't really need that kind of accuracy for cyclomatic complexity.
>
> ...省略
>
> Limitations
>
> This approach makes the Lizard implementation simpler and more focused with partial parsers for various languages. Developers of Lizard attempt to minimize the possibility of soft failures. Hard failures are bugs in Lizard code, while soft failures are trade-offs or potential bugs.

Google翻訳
> このツールは、実際にコードがどれほど複雑であるかではなく、コードがどれほど複雑に見えているかを実際に計算します。このツールが必要になるのは、含まれているすべてのフォルダとファイルを複雑なときに正しく取得するのは非常に難しいためです。しかし、循環的な複雑さのためにそのような精度を実際には必要としません。
>
> ...省略
>
> 制限
>
> このアプローチにより、Lizardの実装はより簡単になり、さまざまな言語のパーシャルパーサーに集中します。 Lizardの開発者は、ソフト障害の可能性を最小限にしようとしています。ハード障害はLizardコードのバグですが、ソフト障害はトレードオフまたは潜在的なバグです。

要約すると

- lizard は CCN を正確には計算しない。
- さまざまな言語の複雑度の計算を簡単な実装で実現するために、パーサーとして実装する。

ぶっちゃけ「まじかー」と思いました。CCN = 「ソースコードの分岐の数」 = 「テストコード件数の参考値」 と考えていたのですが、lizard は CCN を正確には計算しないのです。このポリシーを知ってからは、lizard の出す CCN をテストコード件数の参考値として利用する場合は、ちゃんと考察した方が良いように感じました。

# 私が現状の lizard の出力に疑問をもっていること

いくつかは lizard 本体に issue としてあげましたが、それ以外にもいくつか疑問がありました。GitHub の issue は基本的には英語でコミュニケーションを取る形になっており、私はできれば母国語（日本語）で議論できたらいいなと思い、Qiita に記事としてまとめたいと思いました。コメントいただけると助かります。

## `for` という引数ラベルが分岐として扱われる

- https://github.com/terryyin/lizard/issues/265
- https://github.com/terryyin/lizard/pull/266

これは前述したとおりです。ツールの筆者もこれを感じていています。

私なりの解決法をプルリクで出していますが、前処理でごにょごにょするよりかは、きちんと構文解析の処理を入れた方がいいような気もします（だれか詳しい人助けて欲しい...）

## `guard` 節が分岐として扱われない

- https://github.com/terryyin/lizard/issues/267
- https://github.com/terryyin/lizard/pull/268

コードリーディングの際に感じたことです。`conditions` 変数に `guard` 節がないため、CCN の計測対象になりません。しかしながら、実際には `if` 文と等価であるため、CCN として計上すべきと考えています。

## nil 合体演算子 `??` が分岐として扱われない

- https://github.com/terryyin/lizard/issues/269

上記リンクの通り、nil合体演算子は分岐構造を成すため CCN として計上すべきと考えます（ツールの筆者も同じ考えのようです）。しかしながら、これを計上するには lizard に大きな変更を加える必要があり、lizard のポリシーに反する可能性があります。

また、[下記の記事](https://www.google.com/amp/s/developer.diverse-inc.com/entry/2019/04/29/120000%3famp=1)にあるように、nil合体演算子は CCN を増加させないという認識がコミュニティにありそうだと感じており、「複雑そうに見える箇所を計測する」という lizard のポリシーを当てはめると、議論の余地があると考えています。

- [循環的複雑度を上げないためのSwiftプログラミングTips](https://speakerdeck.com/imaizume/tips-of-swift-programming-to-reduce-code-complexity)

## `if let` の条件をカンマで繋いだ時、分岐として扱われない

`if let` はカンマで複数記述することができます。このとき、各 `let` でアンラップが失敗した場合は `else` 節に分岐します。したがって、このときのカンマは `&&` と等価であり、`&&` を CCN で計上しているのであれば、このカンマも CCN として計上すべきだろう、という論理です。

解決には関数内部の構文解析が必須なので、こちらを対応する場合は引数ラベルの対応も関数内の構文解析に統一した方が良いように感じます。

## ネストした関数が計上されない

lizard はトップレベルの関数単位で CCN が計上されます。そのため、ネストした関数を持つ関数は CCN が大きい傾向にあります。いくつかの言語はネストした関数は分けて計上されるため、swift でも分けた方がいいかなぁと思いました。

一方、ラムダの糖衣構文と考えると単なる代入文なので、そこまで細かく計上しなくても良いかなぁとも思えました。

```swift:これと
func f0() {
  func f1() { ... }
}
```

```swift:これは等価
func f0() {
  let f1 = { () in ... }
}
```


# まとめ

- lizard は CCN を正確には計測しない。
- swift の CCN を「正確に」計測したい場合は lizard 以外のツールを使った方が良い [^1]
- みんなが使ってるツールに貢献できることは面白い。

# 参考文献

- [循環的複雑度 - MATLAB & Simulink - MathWorks](https://jp.mathworks.com/discovery/cyclomatic-complexity.html)
- [循環的複雑度について](https://qiita.com/yut_arrows/items/16749e02313109071338)
- [Mccabe's Cyclomatic Complexity: Calculate with Flow Graph (Example)](https://www.guru99.com/cyclomatic-complexity.html)
- [Otemachi.swift #03 にて「循環的複雑度を上げないためのSwiftプログラミングTips」という発表をしてきました](https://developer.diverse-inc.com/entry/2019/04/29/120000)

[^1]: swiftlint は CCN によって警告を出せるが、アンラップに関しては CCN を計上していない。
