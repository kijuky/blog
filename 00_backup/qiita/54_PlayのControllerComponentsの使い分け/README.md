https://qiita.com/kijuky/items/26e571ea1a73a146149f

---

[Play2.6からコントローラーへのDIのためにControllerComponentsが導入](https://www.playframework.com/documentation/latest/Migration26#Scala-Controller-changes)されました。導入目的に応じて3種類用意されていますので、その使い分けについて解説します。

# 今北産業

- `guice` を使っている場合は`InjectedController`を使う。
- コンパイル時DIを使っており、そのコントローラーのコンストラクタに追加の引数がない場合は`BaseController`を使う。
- コンパイル時DIを使っており、そのコントローラーのコンストラクタに追加の引数がある場合は`AbstractController`を使う。

では解説です。

# InjectedController

プロジェクトでDIに`guice`を使っている場合は、基本的にはこれ一択です。迷う必要はありません。

```scala
class FooController @Inject() () extends InjectedController {
  def foo = Action(parse.json) {
    Ok
  }
}
```

`guice`によって`ControllerComponents`の実装が渡されてきます。

# BaseController と AbstractController

プロジェクトで[コンパイル時DI](https://www.playframework.com/documentation/latest/ScalaCompileTimeDependencyInjection)を使っている場合、`BaseController`/`AbstractController`を使って依存性を注入します。その際の使い分けは[`ControllerComponents`に定義されているフィールド](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/mvc/ControllerComponents.html)以外のインスタンスの注入が必要かどうか、です。

## 追加の引数がない場合はBaseController

追加の引数がない、つまり`ControllerComponents`に定義されているフィールド以外のインスタンスの注入が必要ない場合は`BaseController`を使います。

```scala
class FooController(override val controllerComponents: ControllerComponents)
    extends BaseController {

  // Action and parse now use the injected components
  def foo = Action(parse.json) {
    Ok
  }
}
```

`BaseController`は`ControllerComponents`に定義されているフィールドをそのまま自身のフィールドにプロキシしているので、クラスの中でそのまま使えます。

## 追加の引数がある場合はAbstractController

追加の引数がある、つまり`ControllerComponents`に定義されているフィールド以外のインスタンスの注入が必要な場合は`AbstractController`を使います。

```scala
class FooController(cc: MyControllerComponents)
    extends AbstractController(cc) {

  // Action and parse now use the injected components
  def foo = Action(parse.json) {
    Ok
  }

  // 追加のインジェクションとして`myAction`を使いたい場合
  def bar = cc.myAction(parse.json) {
    Ok
  }
}
```

（※`AbstractController`は`BaseController`を継承しているので、`BaseController`の時と同様に、`ControllerComponents`に定義されたフィールドをクラス内でそのまま利用できます）

ここで`MyControllerComponents`は`ControllerComponents`を継承し、追加のインジェクションフィールドを持つ`trait`です。

```scala
trait MyControllerComponents extends ControllerComponents {
  // ありがちなのは、ログイン処理をするアクションを追加する、とか。
  def myAction: ActionBuilder[Request, AnyContent]
}
```

これを、[ApplicationComponents](https://www.playframework.com/documentation/latest/ScalaCompileTimeDependencyInjection#Application-entry-point)が継承し、`FooController`には`this`を渡します。

```scala
class ApplicationComponents(context: Context)
    extends BuiltInComponentsFromContext(context)
    /* ...その他のComponentsのケーキは省略... */
    with MyControllerComponents {
  
  // インスタンスのワイヤリングは`lazy val`にするのをお忘れなく。。。
  override lazy val myAction: ActionBuilder[Request, AnyContent] = ... /* ここでインスタンスを初期化 */

  // ControllerComponents のためのワイヤリング
  override lazy val actionBuilder: ActionBuilder[Request, AnyContent] = defaultActionBuilder
  override lazy val parsers: PlayBodyParsers = playBodyParsers

  // 説明のためのインスタンス
  lazy val myControllerComponents: MyControllerComponents = this

  // コンポーネントをワイヤリング
  lazy val fooController: FooController = new FooController(myControllerComponents)

  /* ...ルートのワイヤリングは省略 ...*/
}
```

こうすることで、`Controller`へのインジェクションはコンポーネントを渡すだけ、というシンプルなものになります。

### (余談)`cc.myAction`の`cc.`を消したい

いろんなやり方がありますが、Playフレームワークに則るなら`~~Support`traitを用意します。

```scala
trait MyControllerComponentsSupport {
  def cc: MyControllerComponents
  def myAction = cc.myAction
}
```

これをコントローラーにインジェクションします。

```scala
class FooController(override val cc: MyControllerComponents)
    extends AbstractController(cc)
    with MyControllerComponentsSupport {

  // Action and parse now use the injected components
  def foo = Action(parse.json) {
    Ok
  }

  // 追加のインジェクションとして`myAction`を使いたい場合
  def bar = myAction(parse.json) {
    Ok
  }
}
```

これらを徹底することで以下のメリットを得ます。

- コントローラーの引数は常に1つ
- 必要なサポート`trait`は、ケーキパターンによってインジェクトされる。
    - Playの標準の命名規則に沿う(ex: `I18nSupport` など)なので、見た目が揃う。

# 感想

Playのマイグレーション時に気付いたのですが、Playはこの辺の命名に一貫性があって、それに気がつくとレールに乗った開発ができるような気がします。というのも、基本的にケーキパターンを使うため、これらの`Component`はフィールドの命名が揃っています（揃わないとコンパイルエラーになる）。

- `~~Components` : `ApplicationComponents`にケーキパターンで導入する。
- `~~Support` : `Controller`にケーキパターンで導入する。

また、この命名規則を知っていると、[APIドキュメント](https://www.playframework.com/documentation/2.8.x/api/scala/index.html)から必要な`Components`や`Support trait`を探しやすくなります。

- `Components` : https://www.playframework.com/documentation/2.8.x/api/scala/index.html?search=components
- `Support` : https://www.playframework.com/documentation/2.8.x/api/scala/index.html?search=support

# 参考

https://www.playframework.com/documentation/latest/Migration26#Scala-Controller-changes
