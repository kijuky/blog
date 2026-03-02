https://qiita.com/kijuky/items/b72c1e7b0f53c9b88462

---

Environmentが`Any`だったり、あまり複雑でなければ`ZIOAppDefault`を使えば良いですが、`AService & BService & CService` みたいに複雑になると、それを参照するのも手間だったりします。

デフォルトから異なるEnvironmentを使う場合、`ZIOApp`を継承して次のように実装できます。

```scala
import zio.*

object Main extends ZIOApp:
  override type Environment =
    AService & BService & CService & ... // 必要なサービスを列挙
  override val environmentTag: EnvironmentTag[Environment] =
    EnvironmentTag[Environment]
  override val bootstrap: RLayer[ZIOAppArgs, Environment] =
    AServiceImpl.layer ++ BServiceImpl.layer ++ CServiceImpl.layer ++ ... // サービス実装を列挙
  def run: RIO[Environment, Unit] =
    for _ <- MyService() // サービスを使った処理
    yield ()
```

# 参考

https://stackoverflow.com/a/73919987
