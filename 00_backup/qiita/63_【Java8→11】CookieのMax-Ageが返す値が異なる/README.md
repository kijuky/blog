https://qiita.com/kijuky/items/f8ee01609e420c8d1cee

---

日本語資料があまりなかったので。

JavaにはLTS(Long Term Support)というバージョンがあって、現在だとJava8, 11, 17, 21がLTSバージョンであり、サポート期間にあたります（サポート期間はディストリビューションに寄る）。そのため、Java8で動いているサービスというのも結構あります。しかし、サポート期間は無限ではないので、サポート期間のうちに最新のLTSに上げておくと、不安の種が1つ減ります。今回の話題はそんなJavaのバージョンを8から上げた時に見かけた現象を説明します。

# CookieのMax-Age

HTTPでは、サーバーとクライアントの情報のやり取りの機構の一つに[Cookie](https://developer.mozilla.org/ja/docs/Web/HTTP/Cookies)というものがあります。サーバーがクライアントにクッキーを送ると、クライアントは次のリクエストの時に送られたクッキーを送り返すことで、サーバー側はそのリクエストがどのクライアントからきたかなどを把握することができるようになります。いわゆるセッションというやつです。この機能は便利ですが、セッションが不必要に長いと、自分のセッションを他人が使ってしまうことがあったりします。そのためクッキーには有効期限(Expires)が設けられており、その期限を過ぎるとクッキーは破棄されます。クッキーの有効期限には2種類あり、1つは特定の時刻を表すものと、もう一つは残りの秒数を表すものがあります。前者はExpiresで後者はMax-Ageと呼ばれます。

サーバーはクッキーを送る時に[Set-Cookie](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie)と呼ばれる属性をレスポンスヘッダに渡します。JavaではこのSet-Cookieを表すためにHttpCookieを使います。レスポンスヘッダに渡す属性の文字列を解析するparseメソッドを使うことで、インスタンスを作成できます。

```java
jshell> import java.net.*

jshell> HttpCookie.parse("Set-Cookie: value=1")
$1 ==> [value=1]

jshell> $1.get(0).getMaxAge()
$2 ==> -1
```

ExpiresおよびMax-Ageはオプショナルな要素であり、Max-Ageは設定されていなければ-1を返します。

HttpCookie#parseの面白い点の1つに、Expiresを指定すると、Max-Ageはそれに対応する値を返すようになります。

```java
jshell> import java.time.*, java.time.format.*

jshell> HttpCookie.parse("Set-Cookie: value=1; Expires=" + ZonedDateTime.now(ZoneId.of("GMT")).plusDays(1).format(DateTimeFormatter.RFC_1123_DATE_TIME))
$3 ==> [value=1]

jshell> $3.get(0).getMaxAge()
$4 ==> 86399
```

このように、Max-Ageを与えていませんが、Max-Ageが設定されていることがわかります。

# Java8→11でMax-Ageが返す値が異なる

さて、本題です。Java9以降では、Expiresが過去、つまり期限切れのクッキーの場合、Max-Ageは0になります。Max-Ageが0もしくは負数の場合は期限切れを表します。

```java
jshell> HttpCookie.parse("Set-Cookie: value=1; Expires=" + ZonedDateTime.now(ZoneId.of("GMT")).minusDays(1).format(DateTimeFormatter.RFC_1123_DATE_TIME))
$5 ==> [value=1]

jshell> $5.get(0).getMaxAge()
$5 ==> 0
```

※jshellはJava9以降で提供されたため、jshellを起動しているということは、あなたはJava9以上で動作しています。

しかし、Java8以前でこれは経過秒数を返します。期限切れの場合は負数になります。

```scala
% amm
Loading...
Compiling /Users/yasue.kizuki/.ammonite/predef.sc
Welcome to the Ammonite Repl 3.0.0-M0-56-1bcbe7f6 (Scala 3.3.1 Java 1.8.0_362)
@ import java.net._, java.time._, java.time.format._ 
import java.net._, java.time._, java.time.format._

@ HttpCookie.parse(s"Set-Cookie: value=1; Expires=${ZonedDateTime.now(ZoneId.of("GMT")).minusDays(1).format(DateTimeFormatter.RFC_1123_DATE_TIME)}") 
res1: List[HttpCookie] = [value=1]

@ res1.get(0).getMaxAge() 
res2: Long = -86400L
```

この挙動、-1の場合に「Max-Ageが未設定だった」のか「Max-Ageが1秒前に期限切れになった」のかの判断がつきません。これは[バグとして登録されてJava9で対応されています](https://bugs.openjdk.org/browse/JDK-8005068)。そのため、Java8→11のタイミングでこの差分が発現します。

https://github.com/openjdk/jdk/commit/f894a28859acab03ac210eb40246752b8ae28be3

クッキーの期限が未設定なのか、それとも期限切れなのか判断つかないというのは不便ですね。特にクッキーなんてセッションを構築するために使うものですから、扱う情報がセンシティブだとちょっと恐ろしい気もします。でも大丈夫です、Java9で直ってます。Java9は2017年にリリースされたのでかれこれもう6~7年前の話ですね。現在のLTSはJava21なので、みなさんはそんな不具合踏むことはないと思います。...え？私ですか？この調査で昨日の数時間を溶かしました。

# まとめと教訓

この問題はクッキーの有効期限の未設定という状態をその有効範囲内の数値である-1という値で表してしまったことによる不具合でした。もしこれがIntegerならnullを返すことで未設定であることを明示できたかもしれません。異なる状態を1つの値に当てはめてしまうことによる不具合はなかなか取るのはしんどいので、できれば設計段階で取り除けると良いですね。
