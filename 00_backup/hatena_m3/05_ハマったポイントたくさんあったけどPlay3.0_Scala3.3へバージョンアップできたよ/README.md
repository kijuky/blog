https://www.m3tech.blog/entry/2024/05/17/100000

---

こんにちは。エムスリーエンジニアリンググループでScalaとマミさんが好きな安江です。今回は私が所属している製薬企業向けプラットフォームチームのPlay製プロダクトのPlay/Scalaバージョンアップのお話です。当初Play2.8にバージョンアップしていたのですが、その最中にPlay2.9/Play3.0やScala LTSが出たりもしました。最終的にPlay3.0/Scala3.3にバージョンアップできて本番稼働できたサービスもあるので、そのバージョンアップの経緯をご紹介します。

# Play2.8への道のり

2021年の後期から、所属チームでの[Play](https://www.playframework.com/)製サービスのバージョンアップが始まりました。チームには3つのPlay製サービスがあり、その時のバージョンは次の通りでした。

| サービス | Play | Scala | 備考 |
| --- | --- | --- | --- |
| A | 2.7 | 2.11 | サービスBのRESTクライアントライブラリに依存 |
| B | 2.5 | 2.11 | |
| C | 2.3 | 2.11 | |

サービスAは比較的改修が入るため、Playのバージョンも上がっていました。しかし、サービスAはサービスBに依存しており、サービスBがScala2.11だったため、サービスBをPlay2.8に上げなければサービスAのPlayをバージョンアップすることができません。

サービスCはとても安定しており、良くも悪くも改修が入らず、Playのバージョンも2.3で止まっていました。弊社はGitLabを使っていますが、GitLab CIなどの仕組みもなく、Playのバージョンアップ以外にも改善すべき箇所がたくさんありました。

サービスBとCはオンプレで動いており、サーバーのJavaのバージョンなども古いままでした。sbtも古く、依存ライブラリ解決にivyが使われて環境構築に30分かかっていたりしました。とにかくいろんなものが古くて改善のしがいのあるサービスたちが揃っていました。

2023年8月、最終的にこれら3つのサービスはPlay2.8/Scala2.13/Java11/sbt1.7に更新しました。すべてクラウドで動作し、GitLab CIでデプロイできるようになりました。

# Play3.0へのバージョンアップ

2022年9月、Play界隈に衝撃が走ります。Play内部で使っていた[Akka](https://akka.io/)の[ライセンスがBCLに変更](https://www.lightbend.com/blog/why-we-are-changing-the-license-for-akka)されました。Play越しにAkkaを使う場合は特例が適用されることになっていましたが、最終的には[Akkaを使わないPlay3.0の開発](https://github.com/orgs/playframework/discussions/11405)が開始されました。そして2023年11月に[Play3.0がリリース](https://github.com/playframework/playframework/releases/tag/3.0.0)されました。Play3.0はAkkaからフォークした[Pekko](https://pekko.apache.org/)というライブラリを使うようになりました。また、Akkaを使う[Play2系はメンテナンスが停止されることも検討されている](<https://www.playframework.com/documentation/latest/General#End-of-life-(EOL)-Dates>)ようです。

また、2023年5月にはScala初のLTSバージョンである[Scala3.3がリリース](https://scala-lang.org/blog/2023/05/30/scala-3.3.0-released.html)されました。Scalaは3系になってバージョン互換性がかなり向上しましたが、加えて長期サポートバージョンが提供されたことによって、より安定して利用することが可能になりました。

そのため、PlayユーザーとしてはできるだけPlay3.0/Scala3.3にバージョンアップしたいところです。しかしながらその道のりにはハマりポイントがありました。以下でその概要を紹介します。

## ハマり1：依存ライブラリがPlay2系に依存している

Play2系とPlay3系ではgroupId(package)が`com.typesafe.play`から`org.playframework`に変わりました。そのため、ライブラリが特定のPlayのクラスに依存するようになっていると、実行時エラーが発生する可能性があります。

したがって、Playに依存しているライブラリを使用する場合、そのライブラリのPlay3対応が必須になります。しかしながらそのライブラリがPlay2系で更新が止まってしまっている場合は、そのライブラリが依存しているPlayのバージョンを上げるか、そのライブラリを使わずにロジックを変更する必要があります。

今回、社内ライブラリや[memcachedライブラリ](https://github.com/mumoshu/play2-memcached)がPlay2系のままだったので、それに依存するサービスBとサービスCはPlay3.0にバージョンアップできませんでした。サービスAは[json4sライブラリ](https://github.com/tototoshi/play-json4s)を使っていましたが、違うライブラリを使うことで回避しました。結果、サービスAについてはPlay3.0バージョンアップを実施できるようになりました。

## ハマり2：ScalikeJDBCの依存関係

サービスAは[ScalikeJDBC](https://scalikejdbc.org/)を使用していました。PlayとともにScalikeJDBCを使用している場合、Play/Scalaのバージョンの組み合わせが限定されます。具体的には次の表になります。

| Play | Scala | 利用可能なScalikeJDBC |
| ---- | ---- | ---- |
| 2.9 | 2.13 | 3.5 |
| 2.9 | 3.3 | 4.0 |
| 3.0 | 2.13 | なし |
| 3.0 | 3.3 | 4.2 |

「なし」の組み合わせは、PlayとScalikeJDBCが依存する[scala-parser-combinators](https://github.com/scala/scala-parser-combinators)という依存ライブラリのメジャーバージョンが食い違う結果、発生します。scala-parser-combinatorsのバージョンをどちらかに合わせて回避できますが、その場合は一方のライブラリのテストが通っていないことに留意しましょう。

可能な限り検証済みのコードを利用するために、利用可能なScalikeJDBCが使える組み合わせでバージョンアップする計画を立てました。具体的には次のようなフローになりました。

1. Play2.8/Scala2.13/ScalikeJDBC3.5
2. Play2.9/Scala2.13/ScalikeJDBC3.5 (Play2.8 → Play2.9)
3. Play2.9/Scala3.3/ScalikeJDBC4.0 (Scala2.13 → Scala3.3)
4. Play3.0/Scala3.3/ScalikeJDBC4.2 (Play2.9 → Play3.0)

## ハマり3：サーバーバックエンドの変更

Play2系はAkkaベースであり、既存サービスはサーバーバックエンドもAkka HTTPを使っていました。しかし、Play2系が使っているAkkaはScala3対応していません。そのため、Play2.9ではAkka/Akka HTTPをScala3でも動かすためのハックが導入されています。

今回は、そのハックを利用せずにサーバーバックエンドをAkka HTTPから[Netty](https://www.playframework.com/documentation/latest/NettyServer)に変更しました。ただし、Play3.0/Scala3.3のタイミングで[Pekko HTTP](https://www.playframework.com/documentation/latest/PekkoHttpServer)に変更しています。

1. Play2.8/Scala.2.13/Akka HTTP
2. Play2.9/Scala2.13/Akka HTTP
3. Play2.9/Scala3.3/Netty
4. Play3.0/Scala3.3/Pekko HTTP


## ハマり4：sttpのバックエンドの変更

今回のサービスでは外部サービスのREST API呼び出しのために[sttp](https://sttp.softwaremill.com/en/stable/)を利用していました。Play2.9からPlay3.0へバージョンアップするタイミングで、sttpに関連するバグに遭遇しました。バックエンドに`AsyncHttpClientFutureBackend`を使っていたところ、Play3.0だととあるクラス（`com.typesafe.netty.HandlerPublisher`）が見つからない、というエラーが発生しました。

Play+sttpの最小構成プロジェクトで再現できなかったこと、sttpの次期バックエンドでは`HandlerPublisher`が使われないこと、sttpはバックエンドに[Pekko](https://sttp.softwaremill.com/en/stable/backends/pekko.html)も利用できることなどから、この不具合の根本解決はせずに、Pekkoのバックエンドに切り替えることで回避しました。

## ハマり5：if式が値を返さない

Scala2からScala3への変更点の一つに、else句を持たないif式が値を返さないというのがあります。この挙動に依存している場合、特にXMLを組み立てている場合などは注意が必要です。if式全体が`Unit`として解釈されてしまうため、この変更点を知らないと、まるで条件式が`false`になってしまったかと誤読してしまいます。

# まとめ

2024年5月現在、各サービスのPlay/Scalaバージョンは次のようになりました。サービスBとサービスCは依存ライブラリの都合上、Play3.0に上げることはできませんでしたが、Play2.9にまで上げることができました。今後、依存ライブラリやロジックを整理し、サービスBとサービスCもPlay3.0に上げる予定です。

| サービス | Play | Scala | 備考 |
| --- | --- | --- | --- |
| A | 3.0 | 3.3 | サービスBのRESTクライアントライブラリに依存 |
| B | 2.9 | 2.13 | サービスA向けRESTクライアントライブラリのみScala3.3でクロスビルド |
| C | 2.9 | 2.13 | |

Play3.0/Scala3.3という組み合わせは、ライセンスや長期サポートといった面で有利なので、長期間安定したサービス運用を提供したい場合はバージョンアップを検討した方が良いでしょう。その際は依存ライブラリに注意してバージョンアップの計画を立てることが重要になります。

# We are hiring !!

エムスリーでは、サービスの安定運用を通して医療業界に貢献していくことに興味がある仲間を募集しています！

まずはカジュアル面談から、以下URLよりご応募をお待ちしています。

https://jobs.m3.com/engineer/
