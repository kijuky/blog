https://www.m3tech.blog/entry/2024/07/26/100000

---

こんにちは。エムスリーエンジニアリンググループでScalaとマミさんが好きな安江です。今回は私が所属している[デジカルチーム](https://speakerdeck.com/m3_engineering/m3-digikar-engineering-team)のお話です。ZIO Lambdaを使ってScalaネイティブバイナリをAWS Lambdaで動かしてみました。こちらの技術スタックの紹介をします。

<figure class="figure-image figure-image-fotolife" title="ZIO Lambda">[f:id:m3tech:20240726100021p:plain]<figcaption>ZIO Lambda</figcaption></figure>

## 背景

特定の処理で、外部から提供されたJARライブラリを使う必要がありました。弊社電子カルテは[Rails](https://rubyonrails.org/)製なのですが、別のバックエンドに[Skinny](https://skinny-framework.github.io/)(Scala)製のサーバーを使っていたため、そのサーバーに処理を相乗りさせました。しかし、すでに使用していた依存ライブラリのバージョン違いで処理が動かなくなることがわかりました。当時は絶妙なハック[^1]を使って解決していましたが、既存処理が外部JARライブラリによって不安定になること、今回の特定の処理がバッチ的な性格だったことから、その処理を[AWS Lambda](https://aws.amazon.com/jp/lambda/)で再実装することにしました。

## 技術スタック

AWS Lambdaで実装するにあたって、次の技術スタックを使用しました。

### Scala

https://www.scala-lang.org/

ScalaはJavaとの互換性が高く、今回のようなJavaのライブラリを使う際にも適しています。また、[Scala3.3はScalaのLTSバージョン](https://scala-lang.org/blog/2023/05/30/scala-3.3.0-released.html)で、今後も比較的安定して利用できます。

### ZIO

https://zio.dev/

ZIOはScala向けのエフェクトシステムライブラリです。エフェクトシステムとは、副作用の発生（＝環境への作用）を極力後回しにすることで、参照透過的な（＝関数型）プログラミングスタイルを可能にするシステムのことです。より詳しい内容は弊社テックブログの過去記事もあわせてご覧ください。

https://www.m3tech.blog/entry/referential-transparency-effects-in-scala

コーディングの特徴として、クラスやメソッドの名前は一般的なプログラミング言語で使われる命名が優先され、圏論の単語が出てこないように設計されています[^2]。そのため、圏論に造詣が深くないユーザーでも、関数型プログラミングのパワーを得ることができます。

公式サイトにEcosystemがまとまっているのも魅力的です。ZIOを使うことで「どんな課題を解決できるか」に興味がある方は、[その一覧](https://zio.dev/ecosystem/)を見るだけでも参考になると思います。後述するZIO Lambdaも、ZIOのEcosystemを構成する1つです。

あと、ロゴがかっちょいいです。

### ZIO Lambda

https://zio.dev/zio-lambda/

ZIO Lambdaは、ZIOを使ってAWS Lambdaを書くためのライブラリです。ZIO Lambdaは、AWS LambdaのイベントハンドラーをZIOのエフェクトとして扱うことができます。

### GraalVM Native Image

https://www.graalvm.org/jdk21/reference-manual/native-image/

GraalVM Native ImageはJavaプログラムを自己完結型の実行ファイル（ネイティブバイナリ）にコンパイルできます。実行にJVMは必要なく、[distrolessイメージ](https://github.com/GoogleContainerTools/distroless)を利用できます。distrolessイメージはサイズが小さいため、AWS Lambdaのようなユースケースに最適です。

## 実装

これらの技術スタックを使うことで、簡単にAWS Lambdaを実装できます。ぜひそのパワーを感じて欲しいので、実際に動くHello Worldプログラムをご紹介します[^3]。

### プロジェクトの作成

まずはプロジェクトを作成します。

`project/build.properties`
```config
sbt.version=1.10.0
```

後述するマルチステージビルドのために、[sbt-assembly](https://github.com/sbt/sbt-assembly)を追加します。

https://qiita.com/petitviolet/items/0ccdf3d376b482f13c43

`project/plugins.sbt`
```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.2.0")
```

### ライブラリの追加

次に、build.sbtにライブラリを追加します。Scala3.3でビルドするために、ZIO JSONの古いバージョンをevictedします。

https://github.com/zio/zio-lambda/issues/255

`build.sbt`
```scala
lazy val root = project
  .in(file("."))
  .settings(
    name := "hello-world-zio-lambda",
    version := "0.1.0-SNAPSHOT",
    scalaVersion := "3.3.3",
    libraryDependencies ++= Seq(
      "dev.zio" %% "zio-json" % "0.6.2",
      "dev.zio" %% "zio-lambda" % "1.0.4"
    )
  )
  // assembly
  .settings(
    assembly / mainClass := Some("com.example.Handler")
  )
  // publish
  .settings(
    // 既存動作を上書き
    publish := {
      val awsAccount = sys.env("AWS_ACCOUNT")
      val awsRegion = sys.env.getOrElse("AWS_REGION", "ap-northeast-1")
      val registryName = s"$awsAccount.dkr.ecr.$awsRegion.amazonaws.com"
      val appName = name.value
      val repositoryName = s"$registryName/$appName"
      val appVersion = version.value
      val imageTag = s"$repositoryName:$appVersion"
      import scala.language.postfixOps
      import scala.sys.process.*
      s"aws ecr get-login-password --region $awsRegion" #| s"docker login --username AWS --password-stdin $registryName" !!;
      s"docker buildx build --platform=linux/arm64 --provenance=false --tag=$imageTag . --push" !!;
      // クラウドにLambdaを作成済みの場合は以下のコメントを外すことで、Lambdaのコンテナイメージを更新できる
      // s"aws lambda update-function-code --function-name $appName --image-uri $imageTag" !!
    }
  )
```

### ハンドラーの作成

次に、AWS Lambdaのハンドラーを作成します。ZIO LambdaがAWS Lambdaハンドラーの詳細を隠蔽してくれるので、実装は純粋なロジックを書くだけで良いです。

`src/main/scala/com/example/Handler.scala`
```scala
package com.example

import zio.{Task, ZIO, ZIOAppDefault}
import zio.lambda.{Context, ZLambdaRunner}

object Handler extends ZIOAppDefault:
  val handler: (Event, Context) => Task[String] = (event, _) =>
    ZIO.succeed(s"Hello world! ${event.message}")

  override val run: Task[Unit] =
    ZLambdaRunner.serve(handler)
```

イベントはcase classで表現します。

`src/main/scala/com/example/Event.scala`
```scala
package com.example

import zio.json.{DeriveJsonDecoder, JsonDecoder}

final case class Event(message: String)

object Event:
  given JsonDecoder[Event] = DeriveJsonDecoder.gen[Event]
```

### ネイティブバイナリのビルド

ZIO Lambdaは[zipファイルを使った実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-function-zip.html)と、[コンテナイメージによる実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/images-create.html)の2種類を利用できます。両方を試してみたところ、コンテナイメージによる実行のパフォーマンスが良かった[^4]ので、以下ではコンテナイメージによる実行例を紹介します。

[Dockerマルチステージビルド](https://matsuand.github.io/docs.docker.jp.onthefly/develop/develop-images/multistage-build/)を使ってネイティブバイナリをビルドします。ZIO Lambdaではネイティブバイナリを作るために[sbt-native-packager](https://sbt-native-packager.readthedocs.io/en/stable/)を使っていますが、こうすることで、デプロイ処理の中でも処理を共通化できます。

ビルドステージではイメージに[docker-sbt](https://github.com/sbt/docker-sbt)を使います。docker-sbtには`sbt`と`native-image`が同梱された便利なイメージがあります。ビルドには開発環境のファイルをコピーするため、適切な`.dockerignore`を設定することをオススメします。ignore設定は開発環境によって変わるので、ここでは紹介を省きます。

実行ステージではイメージにdistrolessを指定できます。ただし、[glibcを解決するためにdebian12を使用](https://github.com/GoogleContainerTools/distroless/issues/1342#issuecomment-1881646615)します。

`Dockerfile`
```dockerfile
FROM sbtscala/scala-sbt:graalvm-community-21.0.2_1.10.0_3.3.3 AS build
WORKDIR /build
COPY . .
RUN ./build.sh

FROM gcr.io/distroless/base-debian12
WORKDIR /app
COPY --from=build /build/hello-world-zio-lambda ./hello-world-zio-lambda
CMD ["/app/hello-world-zio-lambda"]
```

`native-image`を呼び出すコードをシェルスクリプトに分けることで、CIでも同じ設定でビルドできるようになります。

`build.sh`
```sh
#!/usr/bin/env sh

sbt assembly
native-image \
  -jar target/scala-3.3.3/hello-world-zio-lambda-assembly-0.1.0-SNAPSHOT.jar \
  -o hello-world-zio-lambda \
  --no-fallback \
  --install-exit-handlers \
  --enable-http \
  --link-at-build-time \
  --report-unsupported-elements-at-runtime \
  -H:+UnlockExperimentalVMOptions \
  -H:+StaticExecutableWithDynamicLibC \
  --verbose
```

作成される実行ファイルはstripされていませんが、[`strip`](<https://en.wikipedia.org/wiki/Strip_(Unix)>)してもファイルサイズはさほど変わりません。

### AWS Lambdaへのデプロイ

AWS Lambda用のコンテナーリポジトリをAmazon ECRに作成します。

<figure class="figure-image figure-image-fotolife" title="リポジトリの作成">[f:id:m3tech:20240726100012p:plain]<figcaption>リポジトリの作成</figcaption></figure>

リポジトリにイメージをプッシュします。

```sh
AWS_ACCOUNT=xxxxxxxxxxxx sbt publish
```

ここで、`xxxxxxxxxxxx`はAWSアカウントです。

AWSコンソールでLambdaを新規作成します。自動で実行ロールとCloudWatchロググループを作成してくれます。作成画面では「コンテナイメージ」を選択します。とくにこだわりがなければ、[料金](https://aws.amazon.com/jp/lambda/pricing/)の安いArmでビルドすることをオススメします。今回紹介したマルチステージビルドでは、[Docker Buildx](https://matsuand.github.io/docs.docker.jp.onthefly/buildx/working-with-buildx/)を使ってArmでビルドしています。

<figure class="figure-image figure-image-fotolife" title="Lambdaの作成">[f:id:m3tech:20240726100015p:plain]<figcaption>Lambdaの作成</figcaption></figure>

テストタブからテスト実行できます。コールドスタートだと3〜5秒くらいかかっています。ホットスタートだと数ミリ秒で処理ができています。

| <figure class="figure-image figure-image-fotolife" title="コールドスタート">[f:id:m3tech:20240726100018p:plain]</figure> | <figure class="figure-image figure-image-fotolife" title="ホットスタート">[f:id:m3tech:20240726100010p:plain]</figure> |
| :---: | :---: |
| コールドスタート | ホットスタート | 

AWS Lambdaを更新する時は、build.sbt内のコメントを外して`sbt publish`を実行します。

## まとめ

ZIO Lambdaを使ってScalaネイティブバイナリをAWS Lambdaで動かすことができました。既存アプリとプロジェクトが別れたことで依存ライブラリの不安定要因を取り除くことができました。またインフラ費用を抑えつつ、ネイティブバイナリによる低メモリ使用量と実行速度を手に入れることができました。AWS LambdaやZIOに馴染みがない方は、ぜひこのサンプルを通して関数型プログラミングのパワーを体験してみてください。

# We are hiring !!

エムスリーでは、ScalaやZIOなどを使って医療業界に貢献していくことに興味がある仲間を募集しています！

まずはカジュアル面談から、以下URLよりご応募をお待ちしています。

https://jobs.m3.com/engineer/

[^1]: JARファイルを解凍し、内部に重複するライブラリを削除しました。最終的に、クラスパスに指定した依存ライブラリが1つになるように調整しました。
[^2]: ここは評価が分かれるかもしれません。もしあなたが圏論に馴染みがあるのであれば、むしろ圏論の単語とZIOのメソッドの対応を新たに覚えるコストが発生します。
[^3]: 実際に試す場合はAWSアカウントが必要になります。また、利用状況によっては課金が発生する可能性があります。ご注意ください。
[^4]: 推測ですが、zipファイルの場合は（distrolessではなく）AmazonLinuxが使われるので、その分だけパフォーマンスが落ちた可能性があります。