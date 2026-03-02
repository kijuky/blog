https://qiita.com/kijuky/items/a9e966360f4f40a2b489

---

以前、[PlayをGraalVMで動かす記事](https://qiita.com/kijuky/items/ab3d1aecc9e4466145f9)を書いて、Playが[Guice](https://github.com/google/guice/wiki/Motivation)（ランタイムDI）ベースなことに愚痴ってたら「[コンパイルタイムDIもある](https://www.playframework.com/documentation/2.8.x/ScalaCompileTimeDependencyInjection)よ」とアドバイスを受けたので、試してみました。

https://twitter.com/gakuzzzz/status/1571052080592457729

https://www.playframework.com/documentation/2.8.x/ScalaCompileTimeDependencyInjection

# やってみた

早速ですが、コード差分です。

https://github.com/kijuky/play-on-graalvm/compare/main...feature/compile-time-di

仕事でPlayのバージョンアップ([2.3->2.4](https://www.playframework.com/documentation/2.8.x/Migration24))をやっていて、そこで[DIについて調べることがあった](https://www.playframework.com/documentation/2.8.x/Migration24#Dependency-Injection)ので、コードの修正についてはそこまでつまづくことがなかったです。基本的には下記の通り。

- Guice関連の依存ライブラリを`build.sbt`から外す。
- `@Inject`や`@Singleton`など、ランタイムDI関連のアノテーションを外す。
- 独自の[ApplicationLoader](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/ApplicationLoader.html)を作る。基本的には後述するApplicationComponentを作って`.application`を呼び出す実装にする。こうすると、テスト時にApplicationComponentを再利用できる。
- [BuildInComponentsFromContext](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/BuiltInComponentsFromContext.html)を継承した独自のApplicationComponentを作る。InjectedRoutesGeneratorによって作られたrouter.Routesクラスをインスタンス化するために必要なコントローラーをせっせと作る。[MacWire](https://github.com/softwaremill/macwire/blob/master/README.md)を使わない場合、コンストラクタ引数の順序に注意する。

より詳しい方法はこちら。

https://loicdescotte.github.io/posts/play24-compile-time-di/

https://dvirf1.github.io/play-tutorial/posts/compile-time-dependency-injection/

# 感想

目論見通り、[reflect-config.json](https://www.graalvm.org/22.0/reference-manual/native-image/Reflection/#manual-configuration)からGuice関連のクラスの定義が減っていました。しかし、[akka](https://akka.io/docs/)や[Logback](https://logback.qos.ch/documentation.html)の内部で[Class.forName](https://docs.oracle.com/javase/jp/19/docs/api/java.base/java/lang/Class.html#forName(java.lang.String))を多用しているようで、reflect-config.jsonを完全に無くすことはできませんでした。ただ、今回得られたreflect-config.jsonをPlay本体に加えることで、コンパイルタイムDIを採用している場合にGraalVM化する手間を多少は省けるのではないかなぁと考えています。※ApplicationLoaderだけはどうしてもreflect-config.jsonに追加する必要がある... Play本体を改修して、play.application.loaderのクラスから動的にreflect-config.jsonを作るのはありかもしれない。

Guice版もコンパイルタイムDI版も、結局は同じだけクラスをコンパイルする必要があるので、コンパイル時間は双方そんなに変わりませんでした。Guice版は依存関係をランタイムで解決するのでコンパイルが早いのが特徴ですが、GraalVMとしてビルドしてしまうとそのメリットがなくなってしまいますね。。

また、Guice版は、DI対象が増えるとreflect-config.jsonに追記が必要になりますが、コンパイルタイムDI版はその必要がありません。運用面を考えると、コンパイルタイムDI版の方が手軽かもしれません。

# まとめ

PlayのコンパイルタイムDI版でGraalVM化してみました。必要なreflect-config.jsonが少なくなり、運用面でGuice版より有利になるので、GraalVM化するならコンパイルタイムDI版の方が良さそうです。

最近では、[OpenJDKにはGraalVMが付いてくるらしい](https://www.publickey1.jp/blog/22/openjdkgraalvmopenjdk.html)ですし、[Quarkus](https://ja.quarkus.io/guides/building-native-image)や[Spring Booot](https://spring.pleiades.io/spring-boot/docs/current/reference/html/native-image.html)はGraalVMオプションが標準で付いているので、Playもその波に乗れると嬉しいですね（[マージしてほしい...](https://github.com/playframework/playframework/pull/11371)）。
