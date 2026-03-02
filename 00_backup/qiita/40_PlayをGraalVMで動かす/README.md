https://qiita.com/kijuky/items/ab3d1aecc9e4466145f9

---

ごきげんよう。みなさんScala書いてますか？ところで、今日は[Play](https://www.playframework.com/)を[GraalVM](https://www.graalvm.org/)でビルドしてみたので、その紹介をしたいと思います。サンプルプログラムも貼りますので、気になる方は試してみてください。

# 背景

「Play GraalVM」とかでGoogle検索かけると、過去にそれっぽい健闘をした記事が色々と出てきますが、どれも何がダメだったとか、具体的なコードとかが出てこなかったので、試しに作ってみることにしました。

https://www.playframework.com/blog/play-on-graal

https://scalapedia.com/articles/120/sbt-native-image%E3%81%A7%E3%83%90%E3%82%A4%E3%83%8A%E3%83%AA%E3%82%92%E7%94%9F%E6%88%90%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95

https://qiita.com/petitviolet/items/0ccdf3d376b482f13c43

# 結果

とりあえず、Hello worldとして紹介されている[play-scala-seed.g8](https://github.com/playframework/play-scala-seed.g8)はGraalVMでビルドできました。4点ほど注意点があったので紹介します。

https://github.com/kijuky/play-on-graalvm

## confフォルダ内のファイルを参照できるようにする

まず `PlayKeys.externalizeResources := false` とします。これによりconfディレクトリの中身がjarファイルに含まれるようになり、デフォルトのクラスパス上から`application.conf`とか`logback.xml`などのファイルが参照できるようになります。

https://groups.google.com/g/play-framework/c/hePH94MPQk8

https://github.com/playframework/playframework/pull/4116

次に `resource-config.json` にて動的にロードが必要なファイルを明示します。ただし、独自に `resource-config.json` を書くのは辛いので、後述するagentの出力結果をコピーしましょう。

1点注意すべきところがあります。`resource-config.json` を指定する `-H:ResourceConfigurationFiles` には **絶対パス** を渡すようにしてください。sbtの内部でカレントディレクトリを移動してビルドするため、`build.sbt`からの相対パスだと指定に失敗します。

https://docs.oracle.com/cd/F44923_01/enterprise/21/docs/reference-manual/native-image/Resources/

https://kazuhira-r.hatenablog.com/entry/2019/06/15/222118

## 動的に読み込まれるクラスを参照できるようにする

ここが一番のキモです。GraalVMではリフレクションで使用するクラスは明示的に宣言しておく必要があります。Playはguiceを使っており、guiceが動的ロードを使っているので、guiceでロードされるクラスを全て明らかにする必要があります。これを手動でやるのは億劫なので、動的ロードされたものを記録するagentをアプリに仕込んで、アプリを一通り動かして、その結果を利用するのが一般的です。

サンプルプロジェクトだとこんな感じで結果を取得します:

1. [Dockerfile の 9行目](https://github.com/kijuky/play-on-graalvm/blob/main/Dockerfile#L9)以降をコメントアウトします。
   ```Dockerfile:Dockerfile
   WORKDIR /app
   
   #RUN sbt graalvm-native-image:packageBin
   ...
   ```
2. ビルドします
   ```
   $ docker build .
   ```
3. コンテナに入ってアプリを起動します。
   ```
   $ docker run -p 9000:9000 -it --rm --entrypoint /bin/bash <<ビルドしたimageのsha256>>
   # sbt universal:packageZipTarball
   # cd target/universal/
   # tar -xzvf play-scala-seed-1.0-SNAPSHOT.tgz 
   # cd play-scala-seed-1.0-SNAPSHOT
   # JAVA_OPTS=-agentlib:native-image-agent=config-output-dir=. bin/play-scala-seed
   ```
4. ホスト側から `http://localhost:9000` を叩きます。既存アプリの場合はroutesにあるパスを一通り実行しておくと良いです。
5. コンテナ側で起動したアプリを閉じて、作成されたjsonをプロジェクトにコピーします。
   ```
   # cat resource-config.json
   # cat reflect-config.json
   ```

通常はこれで終わりですが、guiceの場合、追加の修正があります。`GUICE$INVOKERS`に対応するクラスの`queryAllDeclaredConstructors`を`allDeclaredConstructors`に変更する必要があります。

```json:reflect-config.json
  {
    "name":"controllers.Assets",
    "allDeclaredFields":true,
    "queryAllDeclaredMethods":true,
    "queryAllDeclaredConstructors":true}  <- 2. これを allDeclaredConstructors にする
,
  {
    "name":"controllers.Assets$$FastClassByGuice$$17001601",
    "fields":[{"name":"GUICE$INVOKERS"}]}  <- 1. GUICE$INVOKERS があるクラスの
,
```

これが80個近くあります。気合いで直しましょう（もしくはいい感じのjqを書くと良いのかもしれない）。

## ビルドするマシンのメモリを確保する

GraalVMはビルド時に大量のメモリを消費します。もしメモリが足りなかったらエラーコード137を出してビルドが終わってしまいます。サンプルプログラムの場合はとりあえず10GB確保すると安定してビルドできるようになりました。

https://github.com/quarkusio/quarkus/issues/1140

## resourceスキームに対応する

これが多少厄介で、PlayFrameworkの修正が必要になります（あとで[Issue](https://github.com/orgs/playframework/discussions/11253)と[PullRequest](https://github.com/playframework/playframework/pull/11371)出しておこうと思います）。GraalVMは（アプリ内にバンドルされた）外部リソースを`resource://~`というURLで参照するようになります。この`resource`スキームにPlayは対応していないため、（Playにとって）外部リソースを取得するタイミングでエラーが発生します。

https://docs.oracle.com/cd/F44923_01/enterprise/21/docs/reference-manual/native-image/Resources/

修正自体は既存の`bundle`スキームと同じ処理をしてあげれば良いです。

https://github.com/kijuky/playframework/compare/2.8.x...feature/add-resource-scheme-28x

### 余談：修正したPlayのバンドル方法について

この対応していて一番感動したのはここで、Playくらい大きなフレームワークでも、sbtの基本コマンドに忠実に則っているので、かなり楽でした。まずはPlayをcloneしてきます。サンプルプロジェクトだとサブモジュールを張っているので下記コマンドで取得します。

```
$ git submodule update
```

次に、コードを修正した後 `sbt publishLocal` します。

```
$ cd playframework
$ SBT_OPTS=-Dsbt.ivy.home=../ivy2 sbt publishLocal 
```

すると  `2.8.15+3-1e1d609d-SNAPSHOT` みたいなバージョンで ivy2/local にパブリッシュされます。Dockerコンテナでは標準ivyキャッシュディレクトリにコピーします。

```Dockerfile:Dockerfile
COPY ivy2/local/ /root/.ivy2/local/
```

最後に、sbt-pluginのバージョンを、パブリッシュされたバージョンに更新します

```scala:project/plugins.sbt
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.8.15+3-1e1d609d-SNAPSHOT")
```

# 感想

出来上がったファイルは 83MB くらいでした。アプリは 40MB くらいなので、およそ半分がJVMとScalaが占めているんでしょうか？あと起動が早い。レスポンスもJVM版に比べるとかなり良いです（後述）。ネイティブ化によってJVMに依存しなくなったので、簡単にクラウド環境に持っていけるかもなぁとか思いました。

既にSelenium等で全パスを実行するようなテストが整っていると、relect-config.jsonの生成は多少楽になるかな？とか思いました。

# 追記

[パフォーマンス比較](https://github.com/kijuky/play-on-graalvm/tree/standalone-example#%E3%83%91%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%B3%E3%82%B9%E6%AF%94%E8%BC%83)を行いました。サンプルプロジェクトなのであくまで参考値ですが、レスポンスは半分以下、秒間リクエスト数は約1.4倍に向上しました。

| 環境 | 平均レスポンス(ms) | 平均秒間リクエスト数 |
|:-:|:-:|:-:|
| GraalVM | 8.47 | 6.21k |
| JVM | 19.03 | 4.50k |

# 参考

こちらがとても参考になりました

https://qiita.com/TsuyoshiUshio@github/items/c229252f9ecdf577524a
