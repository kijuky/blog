https://qiita.com/kijuky/items/2b92c19bdaaec93c2980

---

Playのコンパイル時DIで用意するBuiltInComponetnsのクラス階層について整理します。

```plantuml

  package play { 
    BuiltInComponents <|-- BuiltInComponentsFromContext

    interface BuiltInComponents {
      +Application application()
      +ApplicationLifecycle {abstract} applicationLifecycle()
      +Configuration {abstract} configuration()
      +Environment {abstract} environment()
    }

    abstract class BuiltInComponentsFromContext {
      -Context context
      +ApplicationLifecycle applicationLifecycle()
      +Configuration configuration()
      +Environment environment()
    }
  }

  circle entrypoint

  package app {
    BuiltInComponents <|.. ApplicationComponents
    ApplicationComponents <|.. ApplicationComponentsFromContext
    BuiltInComponentsFromContext <|-- ApplicationComponentsFromContext
    MyApplicationLoader *.. ApplicationComponentsFromContext
    entrypoint --> MyApplicationLoader

    class MyApplicationLoader {
      +Application load(Context)
    }


    interface ApplicationComponents {
    }

    class ApplicationComponentsFromContext {
    }
  }

  package test {
    BuiltInComponentsFromContext <|-- FakeApplicationComponents
    ApplicationComponents <|.. FakeApplicationComponents

    class FakeApplicationComponents {
    }
  }

  ApplicationComponentsFromContext --|> MemcachedComponents
  interface MemcachedComponents {}

  FakeApplicationComponents --|> CaffeineCacheComponents
  interface CaffeineCacheComponents {}
```

# 基礎

まずはconfに指定したApplicationLoader実装クラス(図では`MyApplicationLoader`)のloadメソッドから始まります。このメソッドの中でBuiltInComponents実装クラス(図では`ApplicationComponentsFromContext`)を使ってApplicationインスタンスを生成します。

```scala:MyApplication.scala
import play.api.Application
import play.api.ApplicationLoader
import play.api.ApplicationLoader.Context

class MyApplicationLoader extends ApplicationLoader {
  override def load(context: Context): Application = {
    new ApplicationComponentsFromContext(context).application
  }
}
```

`ApplicationComponentsFromContext`は`BuiltInComponents`を実装します。`BuiltInComponents`の抽象メンバのうち、`Context`から導出できるものを定義したヘルパークラスが`BuiltInComponentsFromContext`です。

```scala:ApplicationComponentsFromContext.scala
import com.github.mumoshu.play2.memcached.api.MemcachedComponents
import play.api.ApplicationLoader.Context
import play.api.BuiltInComponentsFromContext

class ApplicationComponentsFromContext(context: Context)
    extends BuiltInComponentsFromContext(context: Context)
    with ApplicationComponents
    with MemcachedComponents
```

メインとなるコンポーネント時DIは`ApplicationComponents`が担います。

```scala:ApplicationComponents.scala
trait ApplicationComponents
    extends BuiltInComponents
    with ... /* 他たくさん */ {

  // playはルーターインスタンスを作りさえすれば動く。SIRDを使えば、routesファイルやコントローラーすらなくてもよい。
  override lazy val router: Router = ...
}
```

# テストを考慮した設計
例えば、本番環境ではmemcachedを使い、単体テスト実行時はcaffeineを使う場合、これらのコンポーネントを一緒に使うことはできません。コンポーネントはケーキパターンを想定しているため、同じキャッシュAPIの異なる実装を混ぜることができないのです。

```scala:ApplicationComponents.scala
trait ApplicationComponents
  extends BuiltInComponents
  with MemcachedComponents
  with CaffeineCacheComponents
  ... {

  // この場合 defaultCacheApi を使うと、2つのうち、どれを使えばいいかが決定できない。
  // コンポーネントの実装メンバはlazy valなので、superも使えない。

}
```

そこで、これらを本番とテストで切り替えられるようにクラス構成を考えます。答えとしては既に図で示した通りで、本番では`ApplicationComponentsFromContext`にてmemcachedを実装し、テストでは`FakeApplicationComponents`にてcaffeineを実装します。`FakeApplicationComponents`は本番でも利用する`ApplicationComponents`を継承しているため、キャッシュAPI以外のワイヤリングを再利用できます。やったね。

```scala:FakeApplicationComponents.scala
class FakeApplicationComponents(context: Context)
    extends BuiltInComponentsFromContext(context: Context)
    with ApplicationComponents
    with CaffeineCacheComponents
    with ... {
  ...
}
```

ちなみに、この`FakeApplicationComponents`はこんな感じで使います。

```scala:OneMyAppPerSuite.scala
trait OneMyAppPerSuite extends OneAppPerSuiteWithComponents {
  self: TestSuite =>

  // デフォルトのフェイクコンポーネントを返す
  override def components: ApplicationComponents = {
    new FakeApplicationComponents(context)
  }
}
```

```scala:ControllerSpec.scala
class ClientControllerSpec extends AnyFunSpec with OneMyAppPerSuite {
  describe("/hello") {
    it("コンテンツに「hello」が含まれること") {
      // ここでアプリが起動している
      val path = "/hello"
      val res = route(app, FakeRequest(GET, path)).get
      val contents = contentAsString(res)
      assert(contents.contains("hello"))
    }
  }
}
```
