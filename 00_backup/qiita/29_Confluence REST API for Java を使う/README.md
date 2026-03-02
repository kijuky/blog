https://qiita.com/kijuky/items/eaf207f3daea294511ab

---

みなさん、ご機嫌よう。今日もコンフル使って社内ナレッジの整理ご苦労様でございます。

さて、そんなコンフルエンスですが、REST API の Java ラッパーがあったので、sbtで使ってみた、というレポートです。なんで sbt なのかはこちら。

https://qiita.com/kijuky/items/6e95db63689083de9e55

# はじめに

結論から言うと **REST API をそのまま使う方がいい**です。適切なファサードがないのでリクエストを作るまでが困難ですし、レスポンスから値を得ることもJavaの並列処理に詳しくないといけません。加えて得られるドメインオブジェクトが貧弱で、コストに見合わないかもしれません。

ただし、そこの勉強も含めてちょっと触ってみたいだとか、レスポンスを得るまでの知識はこの記事で紹介するので、そこのコスト削減ができれば、導入も視野に入るかもしれません。

それでは、どうぞ。

# 依存ライブラリの追加

Atlassian 系の Java API は atlassian がホストしているので resolvers に追加しておきます。また、現時点(2021年10月時点)での [confluence-rest-api](https://mvnrepository.com/artifact/com.atlassian.confluence/confluence-rest-client) は 7.13.0 が安定版っぽい感じです。Maven Central を見ると 7.15.0 まで出ているようですが、枝番が理解できなかったので、枝番がない最新版が 7.13.0 です。

```project/plugin.sbt
resolvers ++= Seq(
  "atlassian-public" at "https://packages.atlassian.com/maven/repository/public"
)

libraryDependencies ++= Seq(
  // confluence
  "com.atlassian.confluence" % "confluence-rest-client" % "7.13.0",
  "javax.xml.bind" % "jaxb-api" % "2.3.1" % Runtime,
  "javax.mail" % "javax.mail-api" % "1.6.2" % Runtime
)
```

Java 9 から JAXB 関連のライブラリが削除されたことが影響し、追加で [jaxb-api](https://mvnrepository.com/artifact/javax.xml.bind/jaxb-api/2.3.1) および [javax.mail-api](https://mvnrepository.com/artifact/javax.mail/javax.mail-api/1.6.2) のランタイム時依存が必要になります。

# 認証情報

必要な認証情報は下記になります。とりあえずメモっておきましょう。

| 認証情報 | このページで下記環境変数として参照 |
|:-:|:-:|
| コンフルエンスのベースURL | CONFLUENCE_BASE_URL |
| 接続するユーザー名 | CONFLUENCE_USERNAME |
| 接続するユーザーのパスワード | CONFLUENCE_PASSWORD |

# REST Client の作成

Confluence API では、 REST Client のことを Remote***Service というクラスで管理しています。このクラスを生成することで各種 REST API を叩くことができます。このリモートサービスは以下のインスタンスを必要とします。

## WebResource を提供するプロバイダー

WebResource とは [Jersey](https://eclipse-ee4j.github.io/jersey/) で提供されるオブジェクトのことで、すでに Jersey を知っている人にとっては説明不要かもしれません。雑にいうと接続ホストの情報を保持します。このオブジェクトは [AuthenticatedWebResourceProvider](https://docs.atlassian.com/ConfluenceServer/javadoc/7.13.1/com/atlassian/confluence/rest/client/authentication/AuthenticatedWebResourceProvider.html) から作成できます。

```scala
import com.atlassian.confluence.rest.client.authentication._

lazy val provider = {
  val baseUrl = sys.env.getOrElse("CONFLUENCE_BASE_URL", "")
  val username = sys.env.getOrElse("CONFLUENCE_USERNAME", "")
  val password = sys.env.getOrElse("CONFLUENCE_PASSWORD", "").toCharArray
  val provider = AuthenticatedWebResourceProvider.createWithNewClient(baseUrl)
  provider.setAuthContext(username, password)
  provider
}
```

気になる人は、使用後に close してください。

## リクエストを実行するエグゼキューター

エグゼキューターは知名度高いので知っている人も多いかもしれません。コンカレンシーパッケージにあるアイツです。効率的なのはキャッシュプール使うんでしょうけど、REST APIのサンプル程度に数回叩くんであればシングルスレッドで問題ありません。

```scala
import com.google.common.util.concurrent._

import java.util.concurrent._

lazy val executor =
  MoreExecutors.listeningDecorator(Executors.newSingleThreadExecutor())
```

## 各種リモートサービスを生成

後続の作業のために検索サービスとコンテンツサービスを生成しておきます。リモートサービスは先ほど作成した provider と executor を使えば作れます。

```scala
import com.atlassian.confluence.rest.client._

lazy val contentService: RemoteContentService =
  new RemoteContentServiceImpl(provider, executor)
lazy val searchService: RemoteCQLSearchService =
  new RemoteCQLSearchServiceImpl(provider, executor)
```

# 前提知識

さて、クライアントも得たところで早速リクエストだ！と言いたいところですが、これらのクライアントを使う前に必要な知識が2つあります。CQL と Expansion です。

## CQL(Confluence Query Language)

JQL(JIRA Query Language) の Confluence 版です。JIRA ほど有名ではありませんが、Confluence にも検索用のクエリ言語があります。これを使ってページを特定して、コンテンツを取得します。Confluence には、スペース内に同じタイトルのコンテンツは存在し得ないという制約があるため、スペース名とタイトル名を一意に決めてあげれば、絶対に1つのコンテンツが得られます。その CQL はこのようになります。

```scala
val cql = s"""space = "$space" and title = "$title""""
```

もっと詳細が知りたい人はこちら

https://developer.atlassian.com/server/confluence/advanced-searching-using-cql/

## Expansion

https://docs.atlassian.com/ConfluenceServer/rest/7.13.1/ より引用します。

> How to use expansion in the REST APIs
In order to minimise network traffic from the client perspective, our API uses a technique called expansion.
>
You can use the expand query parameter to specify a comma-separated list of entities that you want expanded, identifying each entity by a given identifier. For example, the value space,attachments requests the expansion of entities for which the expand identifier is space and attachments.
>
You can use the . dot notation to specify expansion of entities within another entity. For example body.view would expand the content body and expand the view rendering of it.

DeepL 翻訳

> REST APIにおけるエクスパンションの使用方法について
REST APIでは、クライアント側のネットワークトラフィックを最小限に抑えるために、expansionという技術を使用しています。
>
expand クエリ・パラメータを使用して、展開したいエンティティのコンマ区切りのリストを指定し、各エンティティを所定の識別子で識別できます。たとえば、space,attachments という値は、expand 識別子が space and attachments であるエンティティの展開を要求します。
>
ドット表記を使用して、他のエンティティ内のエンティティの拡張を指定できます。例えば、body.view は、コンテンツのボディを展開し、そのビューレンダリングを展開します。

つまるところ、REST API で返ってくる情報というのは、ネットワークトラフィックを考慮して最小限に抑えられており、それ以外の情報が欲しい場合は Expansion に指定してね、ということです。残念ながら、コンテンツの何が Expansion を指定しなくても返ってくるのか、Expansion を指定しないと返ってこないのか、その場合の Expansion の指定はどうすればいいのか、がまとまっている資料は見つけられませんでした。

また、API には Expansion の定義があったりなかったりします。例えば `space` というのは [Content.Expansions](https://docs.atlassian.com/ConfluenceServer/javadoc/7.13.1/com/atlassian/confluence/api/model/content/Content.Expansions.html) に定義があります。しかし `body.view` そのものはありません。周辺の API を見る限りだと combine を使って次のように生成するのが想定されているような気もします。

```scala
import com.atlassian.confluence.api.model._
import com.atlassian.confluence.api.model.content._

lazy val bodyView = Expansion.combine(
  Content.Expansions.BODY, 
  ContentRepresentation.VIEW.getRepresentation
)
```

が、たかが `body.view` という文字列を作りたいがだけなのに、これは大袈裟な気もします。`.`で繋げるというドメイン知識を覚える必要はなくなりますが、せいぜいそれくらいのメリットしかない気がします。

ちなみに、`body.view` は実際のページのレンダリング結果を返します。コンテンツの中を解析したい場合はソースコードの方が便利かもしれません。その場合は `body.storage` を利用します（いつも `o` を忘れてしまう...）。

# リクエストする

...さて、多少愚痴っぽくなってしまいましたが、気を取り直して。

ここでは特定のページのソースコンテンツを得ることにしましょう。前述した通り、Confluence ではスペース名とタイトルが決まればページは一意に決定します。

```scala
import scala.collection.JavaConverters._

val contents: Seq[Content] =
  searchService                        //: RemoteCQLSearchService
    .searchContentCompletionStage(cql) //: CompletionStage[SearchPageResponse[Content]]
    .toCompletableFuture               //: CompletableFuture[SearchPageResponse[Content]]
    .get()                             //: SearchPageResponse[Content]
    .getResults                        //: util.List[Content]
    .asScala                           //: mutable.Buffer[Content]
```

軽く説明すると REST Client には *hoge* CompletionStage と呼ばれる API コールメソッドがあります。これは Java API の [CompletionStage](https://download.java.net/java/early_access/panama/docs/api/java.base/java/util/concurrent/CompletionStage.html) と呼ばれるクラスを返します。雑にいうと、並列にしたい処理をメソッドチェーンで書ける、みたいな感じですが、REST API をコールする際にやりたいことなんてほとんどない（本当か？）ので、とっとと結果を得るための Future に変換します。Future はお馴染みなので説明不要ですね。 get メソッドで結果を得ます。検索結果としてのページコンテンツは getResults メソッドで取得します。これは Java の List を返すので、扱いやすいように Scala の Seq(正確には ArrayBuffer) に変換しておきましょう。

得られた Content には id という ContentId 型のプロパティがあります。これを使って contentService から詳細なコンテンツを得ましょう。

```scala
import sbt._

val bodyStroage = Expansion.combine(
  Content.Expansions.BODY, 
  ContentRepresentation.STORAGE.getRepresentation
)

val optContent =
  contents
    .flatMap { content => // 実際には1つしかないはず
      val expansions = Seq(bodyStorage)
      contentService            //: RemoteContentService
        .find(expansions: _*)   //: RemoteContentService.RemoteContentFinder
        .withId(content.id)     //: RemoteContentService.RemoteSingleContentFetcher
        .fetchCompletionStage() //: CompletionStage[Optional[Content]]
        .toCompletableFuture    //: CompletableFuture[Optional[Content]]
        .get()                  //: Optional[Content]
        .asScala                //: Option[Content] <- これは sbt の機能
    }
    .headOption
```

最後の Optional -> Option 変換が地味に便利でした。

さて、ようやく Content が得られたので、ここから詳細なコンテンツを得ることができます。得られる値は xml なので XML オブジェクトに変換すると後で加工するときに楽そうです。

```scala
import scala.xml._

optContent.foreach { content =>
  val body = 
    content
      .getMap
      .get(ContentRepresentation.STORAGE)
      .map(_.getValue)
      .getOrElse("")
  // 得られる body はルート要素が一つではないので、適当なルート要素を用意する
  val bodyWithRoot = s"<root>$body</root>"
  val xml = XML.loadString(bodyWithRoot)
  // ...xml を使って色々する。
}
```

# まとめ

お疲れ様でした。Confluence の Java API を使って REST API を叩いてみました。正直、労力に見合うかはびみょいです... 最後に流れだけを整理しておきます。

- 認証情報付き WebResource プロバイダーを作ります。
- 各種リモートサービスを作ります。
- リクエストします。
    - 前提知識として CQL と Expansion の理解が必要です。
- 得られた詳細コンテンツは XML として解析可能です。

以上です。
