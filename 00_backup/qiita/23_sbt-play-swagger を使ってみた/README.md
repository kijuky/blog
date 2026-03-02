https://qiita.com/kijuky/items/8182d5c5ec43c3ed87e8

---

[Play](https://www.playframework.com/) は [Scala](https://www.scala-lang.org/)/[Java](https://www.java.com/ja/) 向けの Web フレームワークです。このフレームワーク用のプラグインに [sbt-play-swagger](https://github.com/iheartradio/play-swagger) があります。これを使うことで [Play の routes ファイル](https://www.playframework.com/documentation/2.8.x/ScalaRouting)から [swagger.json ファイル](https://swagger.io/specification/v2/)を作成することができます。

swagger.json を用意することで、swagger/openapi のエコシステムが使えます。[各言語向けのクライアントの自動生成](https://swagger.io/tools/swagger-codegen/)や、 [GitLab だと swagger.json をその場でレンダリングしてリッチなUIを提供することができます](https://docs.gitlab.com/ee/api/openapi/openapi_interactive.html)。

# sbt-play-swagger を使う

swagger.json を用意するにはインスペクタを使ったり、アノテーションを使ったりと色々な方法がありますが、ここでは sbt-play-swagger を使った方法を紹介します。

## 使用方法

plugin.sbt に sbt-play-swagger を追加します。

```plugin.sbt
addSbtPlugin("com.iheart" % "sbt-play-swagger" % "0.10.6-PLAY2.8")
```

build.sbt に設定を追加します。

```build.sbt
swaggerRoutesFile := "routes"
swaggerTarget := file("swagger.json")
swaggerDomainNameSpaces := Seq(
  "com.example.domain"
)
swaggerPrettyJson := true
Assets / WebKeys.exportedMappings := Nil
```

| 設定 | 説明 |
|:-:|:-:|
| `swaggerRoutesFile` | routes ファイルの場所。conf/routes であれば指定を省略できる。もし違うファイル名の場合は conf からの相対パスで記述する。 |
| `swaggerTarget` | swagger.json ファイルを出力する場所。こちらはプロジェクトルートからの相対パスで記述する。 |
| `swaggerDomainNameSpaces` | ドメインクラスを検索するパッケージを指定（後述） |
| `swaggerPrettyJson` | pretty 出力するかどうか。基本は git 管理すると思われるので、差分管理しやすい pretty の方がおすすめ |
| `Assets / WebKeys.exportedMappings` | `Nil` にする必要がある |

実行は `swagger` タスクです。

```
$ sbt swagger
```

## メリット

### ドメインクラスから定義を自動生成できる

play の routes にはドメインクラスを利用して型を指定できますが、sbt-play-swagger はその型情報から swagger.json の定義を自動生成できます。そのため、swagger.json の記述をできる限り省略できます。

### swagger.json がそのまま書ける

既存の型をそのまま自動生成してしまった場合（特に Enum など）、想定した swagger.json が出力されないことがあります。その場合は [conf/swagger-custom-mappings.yml を用意する](https://github.com/iheartradio/play-swagger#how-to-override-type-mappings)ことで、自動生成を回避できます。また、そのマッピングよりも優先したい定義があった場合は routes 側でアドホックに定義できます。

### swagger 情報がソースコードに散らばらない

これは好みの問題ですが、特にアノテーション形式で swagger 情報を記述する場合、言語の型情報と swagger の型情報が重複してかなりのノイズになります。加えて swagger のアノテーションは記述が長くなりがちなので、ソースコード中のメタ情報が増えてかなり辛いです。

routes ファイルに記載する場合、swagger の概念的にも場所的に近いので、かなり整理された感じです。ただ、routes ファイルに長大なコメントを書くのはカルチャーショックかもしれません...

## デメリット

### プロジェクト構成に注意

プロジェクト構成によっては swagger.json を作成できないです。特に新規の play プロジェクトで実行できないことがあります。その場合は `Assets / WebKeys.exportedMappings` を `Nil` にする必要があります。

https://github.com/playframework/playframework/issues/5242

### OOM が発生する

場合によってはコンパイル時に OutOfMemoryError が発生します（なんじゃそりゃ）。さすがにそれだと困るので、環境変数が定義されている場合にだけ sbt-play-swagger の設定が有効になるようにしています。

# まとめ

- sbt-play-swagger を使って swagger.json ファイルを自動生成できました。
    - 既存はアノテーションベースだったのですが、swagger 情報を routes に移したので、ソースコードから swagger アノテーションが消えました。
- GitLab による swagger.json のレンダリング機能を使ったので、アプリケーションにバンドルしている swagger-ui を削除できました。
    - swagger-ui はそれなりのコード量あるし、play のバージョンアップによってうまく動かなくなっていたので、整理できてよかった。
- swagger エコシステムを活用していきたい。
    - クライアント自動生成はまだ試してないので、機会があれば試していきたい。
