https://qiita.com/kijuky/items/ca017b1ad1cf2ec6bce6

---

sbtを使ってmaven repositoryにアップロード(publish)するときの、build.sbtに設定する方法をまとめます。

# 今北産業

- 必要なプロジェクトに `publishTo` を設定する。
- 必要なプロジェクトに `publish / skip := false` を設定する。
- 必要ないプロジェクトに `publish / skip := true` を設定する。

# ケースまとめ

## rootがライブラリの場合

`publishTo`を設定します。認証などが不要なら、基本はこの設定だけでアップロードできます。

```build.sbt
publishTo := Some("maven repositoryへのパス")
```

実際にアップロードする場合は`sbt publish`を実行します。

```console
sbt publish
```

## サブプロジェクトの1つがライブラリの場合

例えばWebアプリとAPIクライアントをまとめているプロジェクトを想定します。`root`は`aggregate`とし、**`root`を含む**全てのプロジェクトに対して、アップロードが必要ならば`publish / skip := true`を追加します(root忘れがち)。

```build.sbt
lazy val commonSettings = Seq(
  publish / skip := true,
  ...
)

lazy val publishSettings = Seq(
  publish / skip := false,
  publishTo := Some("maven repositoryへのパス")
)

lazy val root = (project in file("."))
  .settings(commonSettings)
  .aggregate(apiClient, webApp)

lazy val apiClient = (project in file("apiClient"))
  .settings(commonSettings)
  .settings(publishSettings)
  ...

lazy val webApp = (project in file("webApp"))
  .settings(commonSettings)
  ...
```

`commonSettings`でスキップするようにして、後で`publishSettings`でスキップしないようにするのがわかりやすいかなーと思います。sbtのsettingsは宣言的ではなく、書いた通りに実行されるので、`commonSettings`と`publishSettings`を逆にしちゃうと意図通りに動かなくなるので注意です。

## サブプロジェクトの複数がライブラリの場合

WebアプリとAPIクライアントがあって、それらが共通して利用するModelsプロジェクトがある想定です。APIクライアントがModelsプロジェクトに依存するので、Modelsプロジェクトもアップロードできなくてはいけません。

```build.sbt
lazy val commonSettings = Seq(
  publish / skip := true,
  ...
)

lazy val publishSettings = Seq(
  publish / skip := false,
  publishTo := Some("maven repositoryへのパス")
)

lazy val root = (project in file("."))
  .settings(commonSettings)
  .aggregate(models, apiClient, webApp)

lazy val models = (project in file("models"))
  .settings(commonSettings)
  .settings(publishSettings)

lazy val apiClient = (project in file("apiClient"))
  .dependsOn(models)
  .settings(commonSettings)
  .settings(publishSettings)
  ...

lazy val webApp = (project in file("webApp"))
  .dependsOn(models)
  .settings(commonSettings)
  ...
```

注意として、`aggregate`には`models`も含める必要があります。これを含めないと`sbt publish`で`apiClient`しかアップロードされません。しかし、`apiClient`はアップロードされていない`models`に依存するため、これを利用するクライアントは依存関係が解決できなくなります。

# 参考

https://www.scala-sbt.org/1.x/docs/Publishing.html

https://github.com/sbt/sbt/issues/3136
