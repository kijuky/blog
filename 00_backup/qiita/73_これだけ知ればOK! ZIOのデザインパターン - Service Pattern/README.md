https://qiita.com/kijuky/items/a8884ddd2d0ee75c58ba

---

[ZIO](https://zio.dev/)は "Type-safe, composable asynchronous and concurrent programming for Scala" というキャッチコピーで表されるフレームワークです。ZIOには[Service Pattern](https://zio.dev/reference/service-pattern/introduction)と呼ばれるデザインパターンがあり、このパターンによってキャッチコピーにある `Type-safe, composable` (型安全に合成可能) を実現することができます。また、単体テストフレームワークもこのパターンで作られているので、Service Patternに則ることで自然とテストしやすい（＝テスタブルな）構成になります。便利ですね。

# なぜService Patternが必要なのか

そもそも、なぜService Patternが必要なのでしょうか？それはZIOのEnvironment Typeを効果的に使うためのパターンだからです。"Environment Type"とは、`ZIO[R, E, A]`の3つの型引数のうちの一つである`R`のことです。Service Patternは`R`を通して依存性の注入を行う仕組みを提供します。

# Service Pattern

サービスパターンは大きく分けて2人のアクターがいて、「そのサービスを提供する人」と「そのサービスを使う人」です。そして提供側には「サービスの仕様」と「サービスの実装」を用意する必要があります。

## そのサービスを提供する人

### サービスの仕様

標準出力に `Hello` と出力するサービスを作るとしましょう。それを HelloService として、その仕様を記述します。

```scala:HelloService.scala
import zio.*

trait HelloService:
  def hello(): Task[Unit]

object HelloService:
  def hello(): RIO[HelloService, Unit] =
    ZIO.serviceWithZIO(_.hello())
```

本当に必要なのは trait だけですが、object があることで「そのサービスを使う人」が便利になります。

### サービスの実装

そして、この trait を実装するクラスと、この実装を ZLayer として提供する object を作ります。

```scala:HelloServiceImpl.scala
import zio.*

private class HelloServiceImpl extends HelloService:
  override def hello(): Task[Unit] =
    Console.printLine("Hello")

object HelloServiceImpl:
  def layer: ULayer[HelloService] =
    ZLayer(ZIO.succeed(new HelloServiceImpl()))
```

## そのサービスを使う人

サービスを使う場合、実際に使うコードと、そのZIOインスタンスにZLayerを渡す必要があります。

```scala:Main.scala
import zio.*

object Main extends ZIOAppDefault:
  def run = {
    for _ <- HelloService.hello()   // 実際に使っている部分
    yield ()
  }.provide(HelloServiceImpl.layer) // ZLayerを渡している部分
```

ここでの特徴を整理します。

- 「実際に使っている部分」は仕様に依存しています。つまり、実装を入れ替えられます。
- 実装を入れ替えるには「ZLayerを渡している部分」を切り替えます。一般的な使い方としては単体テストです。

面白いのは、ZIOはエフェクトシステムにおける「エフェクト」というインスタンスになります。これは「実際に使っている部分」と「ZLayerを渡している部分」を分離できます。

```scala:Main.scala
import zio.*

object Main extends ZIOAppDefault:
  def domain =
    for _ <- HelloService.hello()          // 実際に使っている部分
    yield ()

  def run =
    domain.provide(HelloServiceImpl.layer) // ZLayerを渡している部分
```

ここで `domain` は仕様にしか依存していない純粋な処理になっています。一方で `run` は依存するインスタンスを注入している部分（一般的に wiring と呼ばれる部分）になります。

サービスパターンは一種のフラクタル構造になっていて、「サービスを提供する人」は、別の「サービスを使用する人」になれます。そのため大きなアプリケーションであっても、小さなサービスの集合体として記述することができます。

また、単体テストは `domain` に対して任意のFakeオブジェクトをprovideすることで処理を切り替えることができます。

### Type-safe

とってつけたように補足を。複雑なdomainを作った際、必要な ZLayer が少なかった場合はコンパイル時点でエラーになり、かつ有効な ZLayer の候補を教えてくれます。Environment Typeの型境界のおかげでいい感じにエラーを生成してくれるんですね。便利。

# まとめ

ZIOには3つの型引数があり、そのうちの一つである `R` を活用するためのデザインパターンがあります。それがService Patternであり、これには次のメリットがあります。

- `R` を通してDIができます。
- サービスの実装ではなく、サービスの仕様に依存します。これにより単体テストがしやすくなります。
- Service Patternはフラクタルです。これを組み合わせることで小さなサービスの集合体から大きなアプリケーションを組み上げることができます。

つまり、ZIOでアプリケーションを作るには、Service Patternだけ知っていればOKなのです。便利。
