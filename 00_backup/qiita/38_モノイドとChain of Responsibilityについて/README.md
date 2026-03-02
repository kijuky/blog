https://qiita.com/kijuky/items/57df6471e5ee7990d65f

---

社内で [Scala with Cats](https://www.scalawithcats.com/dist/scala-with-cats.html) の輪読会をやりました。2章のモノイド・半群を担当し、モノイドと [OOP](https://ja.wikipedia.org/wiki/%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E6%8C%87%E5%90%91%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0) の[デザインパターン](https://ja.wikipedia.org/wiki/%E3%83%87%E3%82%B6%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3_(%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2))である [Chain of Responsibility](https://ja.wikipedia.org/wiki/Chain_of_Responsibility_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3) の類似について紹介しました。

# モノイドとは

「閉じた二項演算」と「単位元」と「結合法則」を持つ集合のことです。これができると fold ができるようになります。単位元を持たないものを半群と呼び、半群のインスタンスは自由に結合(combine)させることができます。

Cats ライブラリでは既知のモノイドを使って足し算をすることができます。

```scala:plus.sc
import $ivy.`org.typelevel::cats-core:2.3.0`
import cats._
import cats.syntax.semigroup._

println(1 |+| 2) // 3
```

# Chain of Responsibility とは

GoF のデザインパターンの一つ。Handler インスタンスの連鎖を作って、処理をその連鎖で実現します。Wikipedia では Logger の例が挙げられています。

```scala:logger.sc
object Logger {
  val ERROR = 3
  val NOTICE = 5
  val DEBUG = 7
}

abstract class Logger(mask: Int) {
  var next: Option[Logger] = None
  def setNext(l: Logger): Logger = {
    next = Some(l)
    this
  }

  def message(msg: String, priority: Int): Unit = {
    if (priority <= mask) {
      writeMessage(msg)
      next.foreach(_.message(msg, priority))
    }
  }

  def writeMessage(msg: String): Unit
}

class StdoutLogger(mask: Int) extends Logger(mask) {
  override def writeMessage(msg: String): Unit = {
    println(s"Writting to stdout: $msg")
  }
}

class EmailLogger(mask: Int) extends Logger(mask) {
  override def writeMessage(msg: String): Unit = {
    println(s"Sending via email: $msg")
  }
}

class StderrLogger(mask: Int) extends Logger(mask) {
  override def writeMessage(msg: String): Unit = {
    println(s"Sending to stderr: $msg")
  }
}

// Build the chain of responsibility
val l: Logger = 
  new StdoutLogger(Logger.DEBUG).setNext(
    new EmailLogger(Logger.NOTICE).setNext(
      new StderrLogger(Logger.ERROR)))

// Handled by StdoutLogger
l.message("Entering function y.", Logger.DEBUG)

// Handled by StdoutLogger and EmailLogger
l.message("Step1 completed", Logger.NOTICE)

// Handled by all three loggers
l.message("An error has occurred.", Logger.ERROR)
```

```:output
Writting to stdout: Entering function y.
Writting to stdout: Step1 completed
Sending via email: Step1 completed
Writting to stdout: An error has occurred.
Sending via email: An error has occurred.
Sending to stderr: An error has occurred.
```

# モノイドによる書き換え

CoR 版は setNext をするところに可変要素があってなんとなくもんにょりしますね。これをモノイドを使って書き換えてみましょう。

```scala:logger2.sc
object Logger {
  val ERROR = 3
  val NOTICE = 5
  val DEBUG = 7
}

// combine を定義しやすくするために trait として定義
trait Logger {
  def message(msg: String, priority: Int): Unit
}

abstract class AbstractLogger extends Logger {
  def mask: Int
  override def message(msg: String, priority: Int): Unit = {
    if (priority <= mask) {
      writeMessage(msg)
    }
  }

  def writeMessage(msg: String): Unit
}

class StdoutLogger(override val mask: Int) extends AbstractLogger {
  override def writeMessage(msg: String): Unit = {
    println(s"Writting to stdout: $msg")
  }
}

class EmailLogger(override val mask: Int) extends AbstractLogger {
  override def writeMessage(msg: String): Unit = {
    println(s"Sending via email: $msg")
  }
}

class StderrLogger(override val mask: Int) extends AbstractLogger {
  override def writeMessage(msg: String): Unit = {
    println(s"Sending to stderr: $msg")
  }
}

// Semigroup type class
import $ivy.`org.typelevel::cats-core:2.3.0`
import cats._
import cats.syntax.semigroup._
implicit val semigroupLogger: Semigroup[Logger] = new Semigroup[Logger] {
  override def combine(l1: Logger, l2: Logger): Logger = new Logger() {
    override def message(msg: String, priority: Int): Unit = {
      l1.message(msg, priority)
      l2.message(msg, priority)
    }
  }
}

// Build the chain of responsibility
val stdoutLogger: Logger = new StdoutLogger(Logger.DEBUG)
val emailLogger: Logger = new EmailLogger(Logger.NOTICE)
val stderrLogger: Logger = new StderrLogger(Logger.ERROR)
val l: Logger = stdoutLogger |+| emailLogger |+| stderrLogger

// Handled by StdoutLogger
l.message("Entering function y.", Logger.DEBUG)

// Handled by StdoutLogger and EmailLogger
l.message("Step1 completed", Logger.NOTICE)

// Handled by all three loggers
l.message("An error has occurred.", Logger.ERROR)
```

今回は単位元を見出せなかったので、半群で実装しています。出力は CoR 版と同じです。

```:output
Writting to stdout: Entering function y.
Writting to stdout: Step1 completed
Sending via email: Step1 completed
Writting to stdout: An error has occurred.
Sending via email: An error has occurred.
Sending to stderr: An error has occurred.
```

# 考察

- next 可変要素がなくなった。
- setNext が |+| になって、括弧のネストが少なくなった。
    - 正確な表現ではない。setNext でも括弧を省略できる。
    - ただ、 |+| という記号だと、括弧の省略が自然に見える。
- 半群を足し合わせるために Logger の型クラスしか用意していないので、一旦 Logger インスタンスとして受けてる。
    - これはジェネリックな畳み込みメソッドを作ってやればよい。
- CoR 版と比べて Logger インスタンスがたくさん生成されている。
    - |+| の数だけ余分な Logger インスタンス生成がある。
    - 気になるときは気になるかもしれない。


# まとめ

OOP の経験が長く、これから Cats やっていきたい人は、CoR をモノイド/半群を使って書き換えてみると面白いかもしれません。

以上です。
