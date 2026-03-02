https://qiita.com/kijuky/items/f1733e0b9e96c8cca015

---

sbt ネタ。業務では slack 通知はよくやるので、これが sbt でできるといろいろ便利。なんで sbt かはこちら。

https://qiita.com/kijuky/items/6e95db63689083de9e55

参照用に。今回使う slack api のリファレンスはこちら。

https://slack.dev/java-slack-sdk/guides/ja/incoming-webhooks

# okhttp 問題

sbt 上で slack api を使う場合、okhttp のバージョンが異なる問題があります。sbt には okhttp3 が載っているのだけど、slack api は okhttp4 が必須です（そのまま呼ぶと実行時エラーになる）。なので、なんらかの方法でこの問題を回避する必要があります。

# 実装

回避方法の1つとしては、 okhttp3 を使って slack api を叩くというものがあります。そのまま叩くのだったら slack api 使う意味ないじゃん、って気もしますが、slack api にはリクエストを構築するためのヘルパーが豊富なので、それが使えるだけでも儲けものです（このリクエストを頑張ってシェルで構築しているものがありますが、JSON のエスケープ地獄で死にそう...）。

```project/plugins.sbt
libraryDependencies ++= Seq(
  "com.slack.api" % "slack-api-client" % "1.12.1"
)
```

```project/SlackUtils.scala
import com.google.gson._
import com.google.gson.FieldNamingPolicy._
import com.slack.api._
import com.slack.api.model.block._
import com.slack.api.model.block.Blocks._
import com.slack.api.webhook._
import okhttp3._
import sbt.io.Using._

import java.io._

object SlackUtils {
  private lazy val slackWebhookUrl = sys.env.getOrElse("SLACK_WEBHOOK_URL", "")
  private lazy val gsonBuilder = new GsonBuilder()
    .setFieldNamingPolicy(LOWER_CASE_WITH_UNDERSCORES)
  private lazy val gson = gsonBuilder.create()
  private lazy val prettyGson = gsonBuilder.setPrettyPrinting().create()

  lazy val slack: Slack = Slack.getInstance()

  implicit class RichSlack(slack: Slack) {
    def send(payload: Payload, debug: Boolean = false): Unit =
      slackWebhookUrl match {
        // for debug
        case empty if empty.trim.isEmpty || debug =>
          println(prettyGson.toJson(payload))
        case webhookUrl =>
          val response = send(webhookUrl, payload)
          if (response.getCode != 200) {
            throw new IOException(response.getMessage)
          }
      }

    def send(webhookUrl: String, payload: Payload): WebhookResponse =
      send(webhookUrl, gson.toJson(payload))

    def send(webhookUrl: String, payload: String): WebhookResponse = {
      val mediaType = MediaType.get("application/json")
      val body = RequestBody.create(mediaType, payload)
      val request = new Request.Builder().url(webhookUrl).post(body).build()
      val newClient = new OkHttpClient().newCall(_: Request).execute()
      resource(newClient)(request)(_.toWebhookResponse)
    }
  }

  implicit class RichResponse(response: Response) {
    def toWebhookResponse: WebhookResponse =
      WebhookResponse
        .builder()
        .code(response.code())
        .message(response.message())
        .body(response.body().string())
        .build()
  }

  implicit class RichPayloadBuilder(payloadBuilder: Payload.PayloadBuilder) {
    def blockSections(sections: Seq[LayoutBlock]): Payload.PayloadBuilder = {
      val blocks = asBlocks(sections: _*)
      payloadBuilder.blocks(blocks)
    }
  }
}
```

環境変数 `SLACK_WEBHOOK_URL` に webhook url を設定します。設定していない場合は単に println します（デバッグ用）。logger info 使っても良かったのですが、出力するのは JSON なので、 `[info] ` が合ってもあまり嬉しくなかったです。。。

```build.sbt
import SlackUtils._
import com.slack.api.model.block.Blocks._
import com.slack.api.model.block.composition.BlockCompositions._
import com.slack.api.webhook.WebhookPayloads._

lazy val helloSlack = taskKey[Unit]("hello slack")
helloSlack := Def.task {
  val text = ":candy: はい、アメちゃん！"
  slack.send(payload(_.text(text)))
}.value

lazy val helloBlockSlack = taskKey[Unit]("block example")
helloBlockSlack := Def.task {
  val text = ":candy: はい、アメちゃん！"
  val sections = Seq(section(_.text(markdownText(text))))
  slack.send(payload(_.text(text).blockSections(sections)))
}.value
```

単にマークダウンを出力したい場合は block を使うと良さげ。また、block を使った場合、 `@display-name` 形式でメンションできます(2021年10月現在)。

こんな感じで、ガッツリ slack api を叩く部分は `SlackUtils` に押し込んであげると、 build.sbt がスッキリします。あんまりやりすぎると柔軟性を損ねるので、slack api のファサードが見えるくらいにして、その補助関数を RichWrapper でにょきにょき生やすと楽しくなります。
