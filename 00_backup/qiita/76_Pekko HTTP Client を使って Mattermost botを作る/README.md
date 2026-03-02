https://qiita.com/kijuky/items/b629f70776a91ef6a288

---

みなさんごきげんよう。今回はPlayFrameworkに搭載されているPekko-HTTPを使って、Mattermost botを作成してみたのでそのしょうかいをしまs。

# はじめに

## Mattermost とは

https://github.com/mattermost/mattermost

Mattermostは、Slackに似たチャットツールです。[Dockerを使ってセルフホストできる](https://docs.mattermost.com/deploy/server/deploy-containers.html)ので、[Synology NAS](https://www.synology.com/ja-jp/dsm/solution/what-is-nas/for-home)を飼っている[逸般の誤家庭](https://anond.hatelabo.jp/20210830231419)では、[Container Manager](https://www.synology.com/ja-jp/dsm/feature/container-manager)を使って無料で[^1]運用することができます。

[Slack無料版だとメッセージが消えたり](https://slack.com/intl/ja-jp/help/articles/27204752526611-Slack-%E3%81%AE%E3%83%95%E3%83%AA%E3%83%BC%E3%83%97%E3%83%A9%E3%83%B3%E3%81%AE%E6%A9%9F%E8%83%BD%E5%88%B6%E9%99%90#u12513u12483u12475u12540u12472u12362u12424u12403u12501u12449u12452u12523u12398u23653u27508)するので、メッセージを残したい場合は選択肢にあるかもです。会社でSlackを使っていて自動化のためにSlack botを作ってみたいが、家で動作確認してみたい場合、家でMattermostを立ててMattermost botを動かしてみるのも良いです。Mattermost botはSlack botとある程度互換性があるようです。

https://zenn.dev/kiyasu7028/articles/af72096f4555cd

## PlayFramework

https://www.playframework.com/

PlayFrameworkは[Scala](https://www.scala-lang.org/)用[Webアプリケーションフレームワーク](https://ja.wikipedia.org/wiki/Web%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%83%95%E3%83%AC%E3%83%BC%E3%83%A0%E3%83%AF%E3%83%BC%E3%82%AF)です。私が得意なので、今回はこれを使います。PlayFrameworkはApache Pekkoを基盤に作られています。

## Apache Pekko

https://pekko.apache.org/

Apache Pekkoは[アクターモデル](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%82%AF%E3%82%BF%E3%83%BC%E3%83%A2%E3%83%87%E3%83%AB)を提供するライブラリです。それをベースにHTTPクライアント機能が提供されています。今回はMattermostサーバーとの通信に[Pekko HTTP](https://pekko.apache.org/docs/pekko-http/current/)を使います。

# Bot作成

PlayFrameworkをベースに作ると、常時起動やPekkoの初期設定をせずに済むので、ロジックに集中できます。

## 0. PlayFrameworkを用意

ScalaをCoursierでインストールしておきます。

https://docs.scala-lang.org/ja/getting-started/install-scala.html#%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF%E3%83%BC%E3%81%AB-scala-%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%81%99%E3%82%8B

適当なフォルダを作ってこれらのファイルを作成します。

```project/build.properties
sbt.version = 1.11.0
```

```project/plugins.sbt
addSbtPlugin("org.playframework" % "sbt-plugin" % "3.0.7")
```

```build.sbt
ThisBuild / version := "0.1.0-SNAPSHOT"

lazy val root =
  project.in(file("."))
    .enablePlugins(PlayScala)
    .settings(
      name := "mattermostbot", // アプリケーションの名前
      scalaVersion := "3.7.0",
      libraryDependencies ++=
        Seq(
          "net.bis5.mattermost4j" % "mattermost-models" % "0.25.0"
        ),
      dependencyOverrides ++=
        Seq(
          // jackson系のバージョンを、Playが使っているものに揃える
          "com.fasterxml.jackson.core" % "jackson-annotations" % "2.14.3",
          "com.fasterxml.jackson.core" % "jackson-core" % "2.14.3",
          "com.fasterxml.jackson.core" % "jackson-databind" % "2.14.3",
          "com.fasterxml.jackson.datatype" % "jackson-datatype-jsr310" % "2.14.3"
        )
    )
```

ログの設定はなんでも良いですが、とりあえず下記をコピペでもOKです。

https://logback.qos.ch/manual/configuration.html#syntax

```conf/logback.xml
<?xml version="1.0" encoding="UTF-8" ?>

<!-- https://www.playframework.com/documentation/latest/SettingsLogger -->

<!DOCTYPE configuration>

<configuration>
    <import class="ch.qos.logback.classic.encoder.PatternLayoutEncoder"/>
    <import class="ch.qos.logback.classic.AsyncAppender"/>
    <import class="ch.qos.logback.core.FileAppender"/>
    <import class="ch.qos.logback.core.ConsoleAppender"/>

    <appender name="FILE" class="FileAppender">
        <file>${application.home:-.}/logs/application.log</file>
        <encoder class="PatternLayoutEncoder">
            <pattern>%date [%level] from %logger in %thread - %message%n%xException</pattern>
        </encoder>
    </appender>

    <appender name="STDOUT" class="ConsoleAppender">
        <encoder class="PatternLayoutEncoder">
            <pattern>%highlight(%-5level) %logger{15} - %message%n%xException{10}</pattern>
        </encoder>
    </appender>

    <appender name="ASYNCFILE" class="AsyncAppender">
        <appender-ref ref="FILE"/>
    </appender>

    <appender name="ASYNCSTDOUT" class="AsyncAppender">
        <appender-ref ref="STDOUT"/>
    </appender>

    <logger name="play" level="INFO"/>
    <logger name="module" level="DEBUG"/>

    <!-- Off these ones as they are annoying, and anyway we manage configuration ourselves -->
    <logger name="com.avaje.ebean.config.PropertyMapLoader" level="OFF"/>
    <logger name="com.avaje.ebeaninternal.server.core.XmlConfigLoader" level="OFF"/>
    <logger name="com.avaje.ebeaninternal.server.lib.BackgroundThread" level="OFF"/>
    <logger name="com.gargoylesoftware.htmlunit.javascript" level="OFF"/>

    <root level="WARN">
        <appender-ref ref="ASYNCFILE"/>
        <appender-ref ref="ASYNCSTDOUT"/>
    </root>

</configuration>
```

今回はサーバー機能を使わないのですが、ビルドのために空の `conf/routes` ファイルを用意しておきます。

```conf:conf/routes
# 中身は空
```

最低限のアプリケーションの設定を書いておきます。

```hocon:conf/application.conf
play.application.loader = BotApplicationLoader
play.filters.hosts.allowed = [ ...省略 ]
play.http.secret.key = ${?PLAY_HTTP_SECRET_KEY}
mattermost.host = ${MATTERMOST_HOST}
mattermost.token-id = ${MATTERMOST_ACCESS_TOKEN}
mattermost.bot-id = ${MATTERMOST_BOT_ID}
```

それぞれの設定の詳細はこちら

- play.application.loader - https://www.playframework.com/documentation/latest/ScalaCompileTimeDependencyInjection#Application-entry-point
- play.filters.host.allowed - https://www.playframework.com/documentation/latest/AllowedHostsFilter#Configuring-allowed-hosts
- play.http.secret.key - https://www.playframework.com/documentation/latest/Deploying#The-application-secret
- mattermost.host - Mattermostの `MM_SERVICESETTINGS_SITEURL` 環境変数 https://docs.mattermost.com/configure/environment-configuration-settings.html#site-url
- mattermost.token-id - [botアカウント作成](https://developers.mattermost.com/integrate/reference/bot-accounts/)時に表示されるトークンID。1回しか表示されないので注意。
- mattermost.bot-id - botアカウントのユーザーID。これはMattermost上で確認できなかった。websocket接続時に表示されるので、それをメモっておく。

今回は[コンパイル時DI](https://www.playframework.com/documentation/latest/JavaCompileTimeDependencyInjection)を使うため、アプリケーションローダーを用意します。ランタイムDIでも良いですが、自分が好きなのでこっちで書きます。

```app/BotApplicationLoader.scala
import play.api.ApplicationLoader.Context
import play.api.{Application, ApplicationLoader, LoggerConfigurator}

class BotApplicationLoader extends ApplicationLoader:
  def load(context: Context): Application =
    // https://www.playframework.com/documentation/latest/ScalaCompileTimeDependencyInjection#Configuring-Logging
    LoggerConfigurator(context.environment.classLoader).foreach:
      _.configure(context.environment, context.initialConfiguration, Map.empty)

    BotComponents(context).application
```

コンパイル時DIで用意するコンポーネントを作成します。

```app/BotComponents.scala
import com.fasterxml.jackson.databind.{DeserializationFeature, ObjectMapper}
import module.{MattermostClient, MattermostClientModule}

import play.api.ApplicationLoader.Context
import play.api.BuiltInComponentsFromContext
import play.api.routing.Router
import play.filters.HttpFiltersComponents
import router.Routes

class BotComponents(context: Context)
    extends BuiltInComponentsFromContext(context)
    with HttpFiltersComponents:
  // https://www.playframework.com/documentation/latest/ScalaCompileTimeDependencyInjection#Providing-a-router
  lazy val router: Router = Routes(httpErrorHandler)

  // jacksonを使ってjsonのシリアライズ/デシリアライズを行います。
  private val objectMapper: ObjectMapper =
    ObjectMapper().configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
  // PekkoHTTPを使ってサーバーとやりとりするクライアントです。
  private val mattermostClient: MattermostClient =
    MattermostClientModule(actorSystem, objectMapper, configuration)
  mattermostClient.connect()
```

## 1. Pekko HTTP WebSocketを使って投稿されたメッセージを取得

Mattermost botでは、投稿されたメッセージをWebSocketを使って取得します。Pekko HTTPを使ってWebSocket接続してみましょう。

```app/module/MattermostClient.scala
package module

trait MattermostClient:
  def connect(): Unit
```

いわゆるWebアプリケーションのクリーンアーキテクチャーにおける「サービス」は、Playでは「モジュール」という形で実装します。ただし、今回はコンパイル時DIなので、この実装自体は不要です。

https://www.playframework.com/documentation/latest/ScalaPlayModules

```app/module/MattermostClientModule.scala
package module

import com.fasterxml.jackson.databind.ObjectMapper
import net.bis5.mattermost.model.{Post, WebSocketEvent, WebSocketEventType}
import org.apache.pekko.actor.ActorSystem
import org.apache.pekko.http.scaladsl.Http
import org.apache.pekko.http.scaladsl.model.{ContentTypes, HttpEntity, HttpHeader, HttpMethods, HttpRequest, StatusCodes}
import org.apache.pekko.http.scaladsl.model.headers.{Authorization, OAuth2BearerToken}
import org.apache.pekko.http.scaladsl.model.ws.{Message, TextMessage, WebSocketRequest}
import org.apache.pekko.stream.Materializer
import org.apache.pekko.stream.scaladsl.{Keep, Sink, Source}
import play.api.Configuration

import scala.concurrent.duration.{DurationInt, FiniteDuration}
import scala.util.{Failure, Success, Try}

class MattermostClientModule(
  actorSystem: ActorSystem,
  objectMapper: ObjectMapper,
  configuration: Configuration
) extends MattermostClient:
  // 接続先ホスト名
  private val host: String = configuration.get[String]("mattermost.host")
  // 接続用のbotトークン（bot作成時に表示される）
  private val token: String = configuration.get[String]("mattermost.token-id")
  // フィルター用のボットID
  private val botId: String = configuration.get[String]("mattermost.bot-id")
  // 認証用のヘッダー
  private val headers: Seq[HttpHeader] = Seq(Authorization(OAuth2BearerToken(token)))
  // 再接続時のディレイ
  private val reconnectDelay: FiniteDuration = 5.seconds

  // pekkoを利用するためのインスタンス
  implicit val system: ActorSystem = actorSystem
  import actorSystem.dispatcher // ExecutionContext

  override def connect(): Unit = ??? // 続く...
```

Pekko HTTPでWebSocket接続する場合、`Flow`を作る必要があります。`Flow`を作るには`Sink`/`Source`を作る必要があります。

https://pekko.apache.org/docs/pekko-http/current/client-side/websocket-support.html#websocketclientflow

```app/module/MattermostClientModule.scala
  override def connect(): Unit = // ...続き
    val wsUrl = s"wss://$host/api/v4/websocket"
    val incoming = Sink.foreach(receiveMessage) // 後述
    val outgoing = Source.maybe[Message] // Mattermost botではSourceを使わない。
    val webSocketFlow = Http().webSocketClientFlow(WebSocketRequest(wsUrl, headers))

    // outgoingを使わないので、第一戻り値は使わない。
    val ((_, upgradeResponse), close) =
      outgoing.viaMap(webSocketFlow)(Keep.both).toMap(incoming)(Keep.both).run()

    // 続く...
```

WebSocketの接続は、まずHTTPでアクセスし、`SwitchingProtocols`が渡ってくれば、晴れて接続が確立します。

```app/module/MattermostClientModule.scala
    // ...続き
    // 特に何か処理する必要はありません。ここでは単にログ出力しています。
    upgradeResponse.onComplete:
      case Success(upgrade)
          if upgrade.response.status == StatusCodes.SwitchingProtocols =>
        println("Connected successfully.")
      case Success(upgrade) =>
        println(s"Connection failed: ${upgrade.response.status}")
      case Failure(ex) =>
        println(s"Failed to connect", ex)

    // 続く...
```

WebSocketはHTTPのような単発の接続ではなく、双方向の常時接続のために使われます。しかしながら接続確立後にクライアントPCの画面を閉じてスリープ後復帰するなどで接続が途切れてしまうことがあります。ここで再接続するには`closed`を監視する必要があります。

```app/module/MattermostClientModule.scala
    // ...続き
    // 接続が途切れたら呼び出される。reconnectDelayだけ待った後、再度接続する。
    closed.onComplete:
      case Success(_) =>
        println("WebSocket closed. Reconnecting...")
        system.scheduler.scheduleOnce(reconnectDelay)(connect())
      case Failure(ex) =>
        println(s"WebSocket error. Reconnecting...", ex)
        system.scheduler.scheduleOnce(reconnectDelay)(connect())

  // 続く...
```

これでサーバーからメッセージが来るたびに`Sink`に指定した`receiveMessage`が呼び出されます。`receiveMessage`を実装してみましょう。受け取ったメッセージがbotをメンションしていた場合、メンション部分を除いた本文をエコーするようにしてみましょう。

```app/module/MattermostClientModule.scala
    // ...続き
  private val receiveMessage: Message => Unit =
    case message: TextMessage.Strict =>
      parseMessage(message.text)
    case message =>
      // non-text message.

  private def parseMessage(text: String): Unit =
    Try(objectMapper.readValue(text, classOf[WebSocketEvent])) match
      case Success(webSocketEvent)
          if webSocketEvent.getEvent == WebSocketEventType.Posted
            && filterMentions(webSocketEvent.getData.get("mentions")) =>
        parsePostedData(webSocketEvent.getData.get("post"))
      case Success(_) =>
        // do not parse
      case Failure(ex) =>
        // json parse error

  private def filterMentions(mentions: String): Boolean =
    Option(mentions)
      .toRight(new NoSuchElementException("\"mentions\" not exist"))
      .toTry
      .map(objectMapper.readValue(_, classOf[Array[String]])) match
      case Success(mentions) if mentions.contains(botId) =>
        // メンションされた場合のみtrue
        true
      case Success(_) =>
        // メンションされなかった
        false
      case Failure(ex) =>
        // json parse error
        false

  private def parsePostedData(post: String): Unit =
    Option(post)
      .toRight(new NoSuchElementException("\"post\" not exist"))
      .toTry
      .map(objectMapper.readValue(_, classOf[Post])) match
      case Success(post) =>
        val message = post.getMessage

        // メンション部分をトリムする
        val trimmedMessage = message.replaceFirst("^(@\\S+\\s+)+", "")
        replyMessage(post, trimmedMessage)
      case Failure(ex) =>
        // json parse error

  // 続く...
```

## 2. REST APIを使ってメッセージを投稿する

`replyMessage`メソッドに文字列が渡されるので、その文字列をそのままおうむ返しします。

https://pekko.apache.org/docs/pekko-http/current/client-side/request-and-response.html

```app/module/MattermostClientModule.scala
  // ...続き
  private val replyMessage: (Post, String) => Unit =
    case (posted, text) =>
      val post = Post(posted.getChannelId, text)
      val json = objectMapper.writeValueAsString(post)
      val request = HttpRequest(HttpMethods.POST, s"https://$host/api/v4/posts", headers, HttpEntity(ContentTypes.`application/json`, json))
      Http()
        .singleRequest(request)
        .onComplete:
          case Success(response) =>
            // 正常終了
          case Failure(ex) =>
            // エラー
```

これで、メンションされるとその本文をおうむ返しするbotを作成できました。デバッグしてみましょう。

```console
% sbt run
[info] welcome to sbt 1.11.0 (Eclipse Adoptium Java 21.0.6)
[info] loading settings for project mattermostbot-build from plugins.sbt...
[info] loading project definition from mattermostbot/project
[info] loading settings for project root from build.sbt...
[info]   __              __
[info]   \ \     ____   / /____ _ __  __
[info]    \ \   / __ \ / // __ `// / / /
[info]    / /  / /_/ // // /_/ // /_/ /
[info]   /_/  / .___//_/ \__,_/ \__, /
[info]       /_/               /____/
[info] 
[info] Version 3.0.7 running Java 21.0.6
[info] 
[info] Play is run entirely by the community. Please consider contributing and/or donating:
[info] https://www.playframework.com/sponsors
[info] 

--- (Running the application, auto-reloading is enabled) ---

INFO  p.c.s.PekkoHttpServer - Listening for HTTP on /[0:0:0:0:0:0:0:0]:9000

(Server started, use Enter to stop and go back to the console...)
```

「9000で待ってるよ」と出たら、別のコンソールから9000を叩きます。

```console
% open http://localhost:9000
```

すると、ビルドが始まってMattermostサーバーに接続されます。

## 3. 本番用にビルドする

今回は[マルチステージビルド](https://docs.docker.com/build/building/multi-stage/)を使って直接dockerイメージを生成します。こうすることで、Container Managerからイメージレジストリを経ずに直接実行することができます。ただし、前バージョンにリバートする場合はリビルドが必要になるので、あくまで個人利用とか社内利用に留めておくと良いでしょう。

```Dockerfile
FROM sbtscala/scala-sbt:eclipse-temurin-21.0.6_7_1.10.11_3.6.4 AS builder
WORKDIR /app
COPY . .
RUN sbt Universal/packageZipTarball && tar -xvzf target/universal/mattermostbot-0.1.0-SNAPSHOT.tgz

FROM eclipse-temurin:21.0.6_7-jre AS app
WORKDIR /app
COPY --from=builder /app/mattermostbot-0.1.0-SNAPSHOT .
EXPOSE 9000
CMD ["bin/mattermostbot", "-Dconfig.file=conf/application.conf"]
```

```compose.yaml
services:
  mattermostbot:
    build:
      context: .
    container_name: mattermostbot
    ports:
      - "${EXPOSE_PORT:-9000}:9000"
    environment:
      - PLAY_HTTP_SECRET_KEY=${PLAY_HTTP_SECRET_KEY}
      - LINE_ACCESS_TOKEN=${LINE_ACCESS_TOKEN}
      - MATTERMOST_HOST=${MATTERMOST_HOST}
      - MATTERMOST_ACCESS_TOKEN=${MATTERMOST_ACCESS_TOKEN}
      - MATTERMOST_BOT_ID=${MATTERMOST_BOT_ID}
    volumes:
      - ${VOLUME_ROOT:-.}/logs:/app/logs
```

この `compose.yaml` で[プロジェクト](https://kb.synology.com/ja-jp/DSM/help/ContainerManager/docker_project)を構築すると、構築と同時にビルド&デプロイが行われます。ビルド時にメモリが必要になるため、物理メモリが少ない(ex: 2GBとか)だと、ハングする可能性があります。可能な限り物理メモリは盛っておくと良いでしょう。

# まとめ

PlayFrameworkにバンドルしているPekko HTTP Clientを使ってMattermost botを作ってみました。個人的にはWebSocketを使ったプログラミングが初めてだったので、常時接続すげー！だったり、常時接続を実現するために再接続を考慮しなくちゃいけなかったりとREST APIとの勝手の違いにびっくりしました。みなさんも興味が湧いたらトライしてみてください。

それでは、よきScalaライフを！


[^1]: 電気代とドメイン代は別
