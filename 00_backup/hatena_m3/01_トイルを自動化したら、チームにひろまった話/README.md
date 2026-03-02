https://www.m3tech.blog/entry/2022/03/29/110206

こんにちは。Scalaとまどマギのマミさんが大好きな安江です。この記事ではとある日常業務のトイルを自動化したこと、その後の広まりについて紹介したいと思います。

## ここにトイルがあるじゃろ？

私が所属するチームでは、コードレビューする際、GitLabのマージリクエスト機能に加えて、JIRAにコードレビュー用の課題を作成するワークフローになっています。業務進捗をJIRAの課題で管理しているので、JIRA上では各人が通常の業務に加えて、どれだけコードレビューを抱えているかを把握できるようになっています。

[f:id:m3tech:20221228220003p:plain]

コードレビューを依頼する時、レビュアはチームメンバーからランダムで選出します。JIRAとSlackが連携していたため、コードレビュー用の課題を作成したタイミングでSlackに自動投稿されるボットの発言のスレッドに、レビュアをランダム選択するカスタムレスポンスを起動することでレビュアを選出していました。

[f:id:m3tech:20221228215959p:plain]

これまでの内容を整理しましょう。私がコードレビューをチームメンバーに依頼する際

1. GitLabでマージリクエストを作成します。
2. JIRAでマージリクエスト用の課題を作成します。
3. Slackにて自動投稿されたボットの発言のスレッドを開き、カスタムレスポンスを起動する文言を投稿します。カスタムレスポンスは稀に私をレビュアに推薦します。セルフレビューは推奨されないため、その際は再びカスタムレスポンスを実行します。
4. 決まったレビュアを、GitLabのマージリクエストとJIRAの課題に設定します。

[f:id:m3tech:20221228215956p:plain]

あなたは凄腕エンジニアなので1日に100個マージリクエストを作ることもあるでしょう…100個までは行かなくても、多い日には1日4,5個の小さなマージリクエストを作ることはあるかもしれません。そのたびに上記の作業でGitLabとJIRAとSlackを行き来していると、仕事した気になるかもしれませんが、この作業自体に本質的な価値はありません。このような作業のことはトイルとして知られており、これは自動化によって状況を改善できる典型的な例です。

## 自動化、その前に

このようなトイルを自動化するにあたり、私は以下の3点を心がけています。

### 簡単に捨てられるようにする

私たちは物理出社からリモートワークになったり、課題管理ツールをRedmineからJIRAに移行したりと、ここ数年を振り返っても大小さまざまなワークフローの変化にさらされています。あるトイルを自動化しても、ワークフローが変わったら、その自動化した作業自体が必要なくなるかもしれません。ワークフローが変わったタイミングで、作った自動化作業を簡単に捨てられるようにしておくと良いです。

### 既存の手作業の仕組みも残す

手作業によるトイルを完全に無くすことは理想ですが、現実は複雑です。私のチームでは数十のリポジトリが管理対象ですが、それらに今回の自動化を適用することはそれ自体がトイルになりかねません。また、自動化処理がエラーにより途中で動作を停止したらコードレビューができない、という状況も困ります。自動化する際は、既存の手作業の仕組みを残しつつ、オプションとして自動化を導入するとスムーズです。

### 最初に一番インパクトがある適用箇所を狙う

多くの人が同じように「めんどくさいなぁ」と思って作業している箇所に自動化を導入するとスムーズです。今回は別チームからもレビュー依頼が来る、多くの人がコミットに参加するリポジトリを対象にしました。

## これをこうして…こうじゃ

今回はGitLab CIのマニュアルタスクを実行することで、コードレビュー課題の作成を自動化してみました。こうすることで、既存の手作業によるフローでも、自動化による処理でも、コードレビュー課題を作成できるようになります。

[f:id:m3tech:20221228215949p:plain]

この結果、レビュー依頼のフローは以下のように変わります。レビューイからの線が少なくなったのがわかります。

[f:id:m3tech:20221228215952p:plain]

実装は、対象のリポジトリがScala/sbtプロジェクトであったため、sbtのタスクとして記述しました。sbtはScalaが書けるので、Java製のRESTクライアントを使うことができます。コード量はちょっと多いですが、作業自体はIntelliJによるサジェストが効いていい感じです。

```scala
// バージョンは2022年3月時点のもの
resolvers += "atlassian-public" at "https://packages.atlassian.com/maven-external/" 
libraryDependencies ++= Seq(
  "org.gitlab4j" % "gitlab4j-api" % "4.19.0",
  "io.atlassian.fugue" % "fugue" % "5.0.0" % Runtime,
  "com.atlassian.jira" % "jira-rest-java-client-core" % "5.2.2"
)
```

```scala
import com.atlassian.jira.rest.client.api.domain.input.IssueInputBuilder
import com.atlassian.jira.rest.client.internal.async.AsynchronousJiraRestClientFactory
import org.gitlab4j.api.GitLabApi
import org.gitlab4j.api.models.MergeRequestParams
import scala.jdk.CollectionConverters._
import scala.util.Random

// 接続情報
lazy val gitlabUrl: String = /* 接続先のGitLab Url */
lazy val accessToken: String = /* GitLabのプロジェクトアクセストークン */
lazy val serverUri: URI = /* 接続先のJIRA Uri */
lazy val username: String = /* JIRAのボットユーザー名 */
lazy val password: String = /* JIRAのボットユーザーのパスワード */
lazy val allReviewers: Seq[String] = /* レビュア候補のGitLabユーザー名のリスト */
lazy val projectKey: String = /* JIRAのプロジェクト名 */
lazy val issueTypeId: Int = /* JIRAの課題タイプID */

// rest client
lazy val gitlab = new GitLabApi(gitlabUrl, accessToken)
lazy val jira = new AsynchronousJiraRestClientFactory()
  .createWithBasicHttpAuthentication(serverUri, username, password)

// task
lazy val createCodeReviewIssue = taskKey[Unit]("コードレビュー課題を作成します")
createCodeReviewIssue := {
  val logger = Keys.streams.value.log
  for {
    // マージリクエストの情報を取得
    projectPath <- sys.env.get("CI_PROJECT_PATH")
    mrIid <- sys.env.get("CI_MERGE_REQUEST_IID").map(_.toInt)
    mr = gitlab.getMergeRequestApi.getMergeRequest(projectPath, mrIid)

    // 既存レビュアを取得
    jql = s"""project = $projectKey and description ~ "${mr.getWebUrl}""""
    reviewIssues = jira.getSearchClient.searchJql(jql).get().getIssues.asScala.toSeq
    reviewers = reviewIssues.map(_.getAssignee.getName) :+ mr.getAuthor.getName

    // レビュアを選定
    reviewerCandidates = allReviewers.filterNot(reviewers.contains)
    _ = logger.info(
      s"reviewer candidates: ${reviewerCandidates.mkString(", ")}"
    )
    reviewer <- Random.shuffle(reviewerCandidates).headOption
  } {
    logger.info(s"reviewer: $reviewer")

    // レビュー課題の作成
    jira.getIssueClient
      .createIssue(
        new IssueInputBuilder(projectKey, issueTypeId)
          .setSummary(mr.getTitle)
          .setReporterName(mr.getAuthor.getName)
          .setAssigneeName(reviewer)
          .setDescription(mr.getWebUrl)
          .build()
      )

    // マージリクエストの更新
    val reviewerId = gitlab.getUserApi.getUser(reviewer).getId
    gitlab.getMergeRequestApi.updateMergeRequest(
      projectPath,
      mrIid,
      new MergeRequestParams()
        .withReviewerIds(Seq(reviewerId).asJava))
  }
}
```

このタスクをGitLab CIのマージリクエスト画面でマニュアル実行できるようにします。

```yaml
review:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
      allow_failure: true
  script:
    - sbt createCodeReviewIssue
```

## その後の広まり

展開後に、:kami: スタンプを2個もらえました。

[f:id:m3tech:20221228220007p:plain]

新卒の永山さん含めたチームメンバーが、他のリポジトリにもこの機能を追加してくれました。私から指示されたわけではなく、自発的に横展開していってくれたのは嬉しかったです。ちなみに、Ruby on RailsのプロジェクトにはRubyで、Node.jsのプロジェクトにはJavaScriptで、Kotlin/SpringBootのプロジェクトにはGradle.ktsで、それぞれ実装してくれました。プロダクトと同じ言語で実装したことにより、この処理のためだけに追加のScalaプラグインなどを導入する必要がありません。また、それぞれの言語のRESTクライアントで実装してもらったので、定義済みModelクラスを用いた、可読性の高いコードになっています。

現在では6つのリポジトリで展開され、これまでに約200回以上利用されました。こちらの取り組みは、トイルの撲滅に一役買ってくれたような気がします。

## We are hiring!

エムスリーでは、日常の業務に潜むトイルを積極的に撲滅し、より価値のある作業時間を増やして、爆速成長できる環境を一緒に作っていく仲間を絶賛募集中です！興味がありましたら下記よりぜひご応募ください。

https://jobs.m3.com/engineer/
