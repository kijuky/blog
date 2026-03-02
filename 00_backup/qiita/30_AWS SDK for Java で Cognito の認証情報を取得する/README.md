https://qiita.com/kijuky/items/fea26b876840e0d450f9

---

メモです。一応取得できたけど、実際の業務では Go 言語で書きそうなので供養。

https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/examples-cognito.html

https://github.com/awsdocs/aws-doc-sdk-examples/tree/main/javav2/example_code/cognito

# 実装

既に aws コマンドで取得したことがある人にとっては、そのまま Java 言語に翻訳、と言う感じ。ここでは sbt で実装する。なんで sbt かはこちら

https://qiita.com/kijuky/items/6e95db63689083de9e55

```project/plugins.sbt
libraryDependencies ++= Seq(
  "software.amazon.awssdk" % "cognitoidentity",
  "software.amazon.awssdk" % "cognitoidentityprovider"
).map(_ % "2.17.62")
```

bom が提供されていますが、バージョン揃えるだけならこんな書き方ができます。さすが scala、と言う感じ。bom についてはこちら

https://qiita.com/syogi_wap/items/432bbdbe9892eb05e122

```bash:.envrc
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export CLIENT_ID=...
export USER_PASSWORD=...
export USER_POOL_ID=...
export USERNAME=...
```

認証情報は適当に。

```build.sbt
import software.amazon.awssdk.regions._
import software.amazon.awssdk.services.cognitoidentityprovider._
import software.amazon.awssdk.services.cognitoidentityprovider.model._

import scala.collection.JavaConverters._

lazy val awsAccessKeyId: String = sys.env.getOrElse("AWS_ACCESS_KEY_ID", "")
lazy val awsSecretAccessKey: String = sys.env.getOrElse("AWS_SECRET_ACCESS_KEY", "")
lazy val clientId: String = sys.env.getOrElse("CLIENT_ID", "")
lazy val userPassword: String = sys.env.getOrElse("USER_PASSWORD", "")
lazy val userPoolId: String = sys.env.getOrElse("USER_POOL_ID", "")
lazy val username: String = sys.env.getOrElse("USERNAME", "")

lazy val printIdToken = taskKey[Unit]("")
printIdToken := Def.task {
  sys.props ++= Map(
    "aws.accessKeyId" -> awsAccessKeyId,
    "aws.secretAccessKey" -> awsSecretAccessKey
  )
  val cognitoIdp = CognitoIdentityProviderClient // closeable なので、気になる人は Using とか使うと良さげ
    .builder()
    .region(Region.AP_NORTHEAST_1)
    .build()
  val request = AdminInitiateAuthRequest
    .builder()
    .userPoolId(userPoolId)
    .clientId(clientId)
    .authFlow(AuthFlowType.ADMIN_USER_PASSWORD_AUTH)
    .authParameters(
      Map(
        "USERNAME" -> username,
        "PASSWORD" -> userPassword
      ).asJava
    )
    .build()
  val response = cognitoIdp.adminInitiateAuth(request)
  val idToken = response.authenticationResult().idToken()
  println(idToken)
}.value
```

```
$ sbt printIdToken
```

実装して気づいたメモ

- accessKeyId と secretAccessKey の読み出し順序で一番優先度が高いのはシステムプロパティに設定されているもの。
- authFlow は文字列でもいける。でも折角なら型でサジェストされた方が安心感ある。
- Java API なので、Scala の Map 使った場合は `asJava` が必要
- CLI だと `--query` で指定していたものは普通にプロパティからアクセスできる。
