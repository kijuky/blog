https://qiita.com/kijuky/items/22f49792181bc50d96b1

---

zio-lambda を使うと、GraalVMでnative-imageを作成し、それをaws lambdaで呼び出せるようになります。便利ですね。

https://zio.dev/zio-lambda/

ただ、ドキュメント等が若干古いので、2024年におけるサンプルを下記に示します。

# 下準備

下記を用意します。

```properties:project/build.properties
sbt.version=1.9.9
```

```scala:build.sbt
lazy val root = (project in file("."))
  .settings(
    organization := "com.hoge",
    name := "zio-lambda-test",
    scalaVersion := "3.3.3",
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio-json" % "0.6.2",
      "dev.zio" %% "zio-lambda" % "1.0.4"
    )
  )
  // ネイティブビルド関連の設定
  .enablePlugins(GraalVMNativeImagePlugin)
  .settings(
    GraalVMNativeImage / mainClass := Some("com.hoge.SimpleHandler"),
    GraalVMNativeImage / containerBuildImage := Some("ghcr.io/graalvm/native-image-community:21.0.2"),
    graalVMNativeImageOptions := Seq(
      "--verbose",
      "--no-fallback",
      "--install-exit-handlers",
      "--enable-http",
      "--link-at-build-time",
      "--report-unsupported-elements-at-runtime",
      "-H:+UnlockExperimentalVMOptions",
      "-H:+StaticExecutableWithDynamicLibC",
      "-H:+RemoveSaturatedTypeFlows"
    )
  )
```

```scala:project/plugins.sbt
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.9")
```

設定は[zio-lambdaのbuild.sbt](https://github.com/zio/zio-lambda/blob/master/build.sbt#L87-L113)から拝借しています。zio-lambdaとしてはnative-imageの作成方法について指定していないので、[sbt-native-image](https://github.com/scalameta/sbt-native-image)でも構いません。

Scalaは最新LTSの3.3.3を使います。そのためにはzio-jsonの最新バージョンを使う必要があります（詳細は下記issueを参照）。

https://github.com/zio/zio-lambda/issues/255

ドキュメントではgraalvm-ceを使っていましたが、`native-image`を使えるdockerイメージが用意されたので、containerBuildImageにはnative-imageを直接指定しています。

# ソースコード

とりあえずHello Worldを書きます。

```scala:src/main/scala/com/hoge/SimpleHandler.scala
package com.hoge

import zio.Console._
import zio._
import zio.lambda._

object SimpleHandler extends ZIOAppDefault {

  val app = (event: CustomEvent, _: Context) =>
    for {
      _ <- printLine(event.message)
    } yield "Handler ran successfully"

  override val run =
    ZLambdaRunner.serve(app)
}
```

```scala:src/main/scala/com/hoge/CustomEvent.scala
package com.hoge

import zio.json._

final case class CustomEvent(message: String)

object CustomEvent {
  implicit val decoder: JsonDecoder[CustomEvent] = DeriveJsonDecoder.gen[CustomEvent]
}
```

```scala:src/main/scala/com/hoge/CustomResponse.scala
package com.hoge

import zio.json._

final case class CustomResponse(message: String)

object CustomResponse {
  implicit val encoder: JsonEncoder[CustomResponse] = DeriveJsonEncoder.gen[CustomResponse]
}
```

実際には`CustomResponse`は使われていませんが、とりあえずサンプルにあったのでコピっておきます。

# ビルド

ネイティブイメージを作成します。

```shell
sbt GraalVMNativeImage/packageBin
```

## zipファイルでデプロイする場合

bootstrapファイルを作成します。ファイル名は`bootstrap`以外にすることはできません。

```sh:target/graalvm-native-image/bootstrap
#!/usr/bin/env bash

set -euo pipefail

./zio-lambda-test
```

zipに固めます。

```shell
cd target/graalvm-native-image
zip upload.zip bootstrap zio-lambda-test
```

固めたupload.zipをaws lambdaにアップロードすると、実行できます。

## dockerイメージに固める場合

先にaws ecrにアップロード先のリポジトリを作っておきます。

次にdockerイメージをビルドしてプッシュします。

```Dockerfile
FROM gcr.io/distroless/base-debian12
COPY target/graalvm-native-image/zio-lambda-test /app/zio-lambda-test
CMD ["/app/zio-lambda-test"]
```

ベースイメージは`base-debian12`にする必要があります。

https://github.com/zio/zio-lambda/issues/256

```shell
pass=$(aws ecr get-login-password --region us-east-1) 
docker login --username AWS --password $pass <your_AWS_ECR_REPO>   
docker tag native-image-binary <your-particular-ecr-image-repository>:<your-tag>
docker push <your-particular-ecr-image-repository>:<your-tag>
```

プッシュしたdockerイメージを指定してaws lambdaを作成すると、実行できます。

# 注意点

native-imageでビルドしているため、アーキテクチャがローカルPCのアーキテクチャに依存しています。そのため、lambdaのアーキテクチャはローカルPCのアーキテクチャと一致させる必要があります。

# 追記

上記内容をg8テンプレートで作りました。サクッとラムダ書いてみたくなった方、どうぞ。

```shell
sbt new kijuky/zio-lambda.g8
```

https://github.com/kijuky/zio-lambda.g8

# 追記2

変更内容がマージされました。

https://github.com/zio/zio-lambda/pull/257
