https://qiita.com/kijuky/items/6e95db63689083de9e55

---

みなさん、Scala 好きですか？はい、大好きですね、知ってますよ。

ところで、そんな Scala 大好きなみなさんは普段 Scala 書いてますか？え、プロジェクトで使っているのは別言語だから使ってないですって？あるあるですね。ちょっとお試しに Scala 使ってみたいけど環境構築だるいしなぁ... って人も多いかと思います。

そんなあなたに朗報です！Scala には sbt という、すごい便利なツールがあるのです。これを使うと、2つのボイラープレートを許すことで Scala を日常のツールにすることができます！早速みていきましょう。

# 2つのボイラープレート

ボイラープレートとは、一種のテンプレートのことであり、その中でも「意識せずにコピペするようなコード片」のことによく使われます。C言語の `#include <stdio.h>` なんかが有名ですね。「おまじない」とも呼ばれるやつです。

私たちが Scala を日常のツールとして使うために、sbt のルールのうち、2つを覚える必要があります。それが**キー**と**タスク**です。

## キー

sbt における「キー」とは、gradle のタスク名、make のターゲット名のことと考えることができます。sbt の場合、 `sbt compile` とかを実行すると思いますが、「キー」は `compile` の部分です。それでは早速テキストエディタを開いて下記を書いてみましょう。

```scala:build.sbt
lazy val hello = taskKey[Unit]("this is hello task")
```

これは `hello` というキーを用意しました。キーを用意しただけでは処理は走りません。処理を定義するのは次で紹介するタスクが担います。

## タスク

それではタスクを定義しましょう。例に倣って `"hello world"` を出力してみましょう。

```scala:build.sbt
lazy val hello = taskKey[Unit]("this is hello task")
hello := Def.task {
  // これがタスク本体
  println("hello world")
}.value
```

おまじないの量が多くなってしまいました... ただし、おまじないはこの**3行だけ**です（よくタスク定義の `.value` を忘れる...）。これさえ覚えれば Scala を日常のツールとして使えるようになるので少々我慢してください...

（実際に Def.task もマクロなので、sbt 利用者にとっては本当に「おまじない」ですね）

それでは、実行させてみましょう。下記はmacで、コードをクリップボードに保存した状態で実行しました。

```console
$ mkdir test
$ cd test
$ ls
$ pbpaste > build.sbt
$ sbt hello
WARNING: A terminally deprecated method in java.lang.System has been called
WARNING: System::setSecurityManager has been called by sbt.TrapExit$ (file:~/.sbt/boot/scala-2.12.14/org.scala-sbt/sbt/1.5.5/run_2.12-1.5.5.jar)
WARNING: Please consider reporting this to the maintainers of sbt.TrapExit$
WARNING: System::setSecurityManager will be removed in a future release
[info] Updated file ~/test/project/build.properties: set sbt.version to 1.5.5
[info] welcome to sbt 1.5.5 (Homebrew Java 17)
[info] loading global plugins from ~/.sbt/1.0/plugins
[info] loading project definition from ~/test/project
[info] loading settings for project test from build.sbt ...
[info] set current project to test (in build file:~/test/)
hello world
[success] Total time: 0 s, completed 2021/10/09 8:17:03
```

ゴチャゴチャ出力していますが、最後から 2 行目で `hello world` が出ていますね。順調です！

# 日常のツールへ

さて、晴れて私たちは sbt でカスタムタスクを実行できるようになりました。これ自体はいろんなWebページや書籍にも載っているので新鮮味はあまりなかったかと思います。

本題はここからです。

## ある風景

ところで私たちは普段どんなサービスを利用しているでしょうか？業務だと git や gitlab, jenkins, slack, jira, confluence, google spreadsheet なんかはよく使っていますかね？プライベートだと twitter とか line なんかもありますかね。これらを使った定型作業はプログラマである私たちはいろんなツールを駆使して自動化しているかと思います。これらの自動化作業、実は sbt/scala を使うととても便利なんです。

では、手始めに git のブランチリストでも取得してみましょう。例えばあなたがあるプロジェクトのリリース担当なら、ブランチリストを取得してどうのこうの... みたいな作業はよくある話かと思います。さて、おもむろに `git ブランチ リスト` とググると、 `git branch` コマンドを見つけます。

```console
$ git branch
* develop
  main
```

今いるブランチが `*` で表示されています。この出力からブランチリストを得るためには `*` を除く必要がありそうです。

```console
$ git branch | tr -d '*' | xargs
develop main
```

大体良さそうです。ところで、これはローカルにあるブランチしかリストされていません。リモートにあるみんなの作業ブランチを見たい場合はどうでしょうか？

```console
$ git branch -r
  origin/HEAD -> origin/develop
  origin/develop
  origin/master
```

...さて、この時点でUNIXコマンドに詳しくない私は辛くなってきましたが、皆さんはどうでしょうか？シェルで様々な作業を自動化できるメリットは十分承知しているつもりですが、全てが文字列ストリームなので、リスト程度のオブジェクトを表現するのもなかなか辛いものがあります。今回はブランチ名なので、そこに空白がないことは明確なので扱いは楽でしたが、空白がない場合のリストはどう表現しましょうか？要素をダブルクォーテーションで区切ったり、デリミタを違うもの（改行とか）を使ったりがありますが、そろそろヤクの毛刈り感が出てきた気がしませんか？

今回、gitを例に挙げましたが、多くのサービスのAPIはそのサービスの複雑なオブジェクトを表現するためにJSONを使用しています。これを扱おうとするとjqを使うことが多いと思いますが、私はjqのクエリを全て把握できていません。こんな状況下では、「期限切れのJIRAのチケットをユーザーごとにSlackに通知する」みたいな作業を書こうと思うと日が暮れてしまいますね。

## サービスの Java API を使う

さて、これを sbt/scala を使って解決したいと思います。ここでの重要なコンセプトは、この章のタイトルにもなっているように「サービスのJava APIを使う」のです。最初にサービスAPIの知識を習得する必要がありますが、その後は Scala の力を使ってボイラープレートを消滅させるのです。これが「日常のツール化」の第一歩です。

では、先ほどのブランチのリストの取得の例をやってみましょう。sbt で jgit を依存させて、処理を書いていきます。多くの人は jgit ネイティブではないので「なるほどこう書くのね」程度の理解で構いません。

```scala:project/plugins.sbt
libraryDependencies ++= Seq(
  // git
  "org.eclipse.jgit" % "org.eclipse.jgit" % "5.13.0.202109080827-r"
)
```

```scala:build.sbt
lazy val remoteBranchList = taskKey[Unit]("this is remote branch list task")
remoteBranchList := Def.task {

  import org.eclipse.jgit.api._
  import scala.collection.JavaConverters._
  val branchList = Git.open(file(".")).branchList().setListMode(ListBranchCommand.ListMode.REMOTE).call().asScala.map(_.getName)
  println(branchList)

}.value
```

```console
$ sbt remoteBranchList
  ...省略...
ArrayBuffer(refs/remotes/origin/develop, refs/remotes/origin/main) 
```

実際はリモートブランチリストを取得して終わる作業は稀であるため、これをいい感じに関数にしておきましょう。


```scala:build.sbt
import GitUtils._

lazy val remoteBranchList = taskKey[Unit]("this is remote branch list task")
remoteBranchList := Def.task {

  val remoteBranchNames = remoteBranches().map(_.name)
  println(remoteBranchNames)

}.value
```

```scala:project/GitUtils.scala
import org.eclipse.jgit.api._
import org.eclipse.jgit.lib._
import scala.collection.JavaConverters._
import sbt._

object GitUtils {
  lazy val git = Git.open(file("."))
  def remoteBranches() = branches(ListBranchCommand.ListMode.REMOTE)
  def branches(listMode: ListBranchCommand.ListMode) =
    git.branchList().setListMode(listMode).call().asScala

  implicit class RichRef(ref: Ref) {
    def name = ref.getName
  }
}
```

build.sbt 側が、本質を表している感じでいいですね。上記の例で、

- サービスの Java API を使うことで、そのサービスのドメインオブジェクトをすぐに手に入れられる。
    - project/plugins.sbt に依存を書けば maven central を手中に入れられる。
- そのサービスのドメイン知識を Scala を使って隠蔽できる。
    - biuld.sbt が雑多な処理で埋もれにくくなる。
    - 拡張メソッド(implicit class)を使うと get~ が消えて、より Scala っぽくなる。

が理解できたかと思います。

## JIRA の例

もう少し実践的な例も見てみましょう。こちらは現在私が運用しているものです。

前提として、業務で [JIRA](https://www.atlassian.com/ja/software/jira) を使っており、こちらのチケット（課題）のうち、別のツールを使って自動生成されるチケットがあります。このツールによって生成されたチケットの報告者はCIツールのユーザーになっており、真のチケットの報告者にはなっていませんでした。真の報告者はチケットの説明欄に記載があるのですが、JIRA には報告者のフィールドがあるので、これを真の報告者に設定することで JQL などの利用がしやすくなるはずです。ひとまずそれを正しましょう。

そうして書いたコードがこちらになります。

```scala:build.sbt
lazy val fixNoreplyReporterIssues =
  taskKey[Unit]("fix noreply reporter issues")
fixNoreplyReporterIssues := Def.task {

  val issues = jira.issues(filterId = 13657)
  issues.foreach { issue =>
    """\* 依頼者: (.+?)@m3\.com""".r
      .findFirstMatchIn(issue.description)
      .map(_.group(1))
      .foreach(issue.reporterName = _)
  }

}.value
```

コードを短くするために、いろいろな Utils クラスも用意しましたが、ここで言いたいことは「build.sbt に書いてある処理が、ほぼ思考の通りである」というところです。コメントを追加するとこんな感じ：

```scala
  val issues = jira.issues(filterId = 13657) // filterId が 13657 のチケットを全て取ってくる。このフィルターはCIツールのユーザーによって作成された、報告者が "noreply" となっている課題をフィルターする。
  issues.foreach { issue =>                  // それぞれのチケットに対して、
    """\* 依頼者: (.+?)@m3\.com""".r          // 所定の正規表現で、
      .findFirstMatchIn(issue.description)   // チケットの説明(description)をマッチさせ、
      .map(_.group(1))                       // メールアドレスの先頭部分をとってきて、
      .foreach(issue.reporterName = _)       // それをチケットの報告者(reporter)に設定する。
  }
```

この読み方に慣れてくると、プログラムがどんどん読みやすくなります（もっと攻めるなら、「チケットの説明から報告者をとってくる」処理は、関数化しても良かったかもしれません）。API を使ったことで、チケット（Issue）や報告者（Reporter）は適切なオブジェクトがあるため、この処理の仕様変更が来た際も、IDE の支援により、シェルスクリプトなどを変更するよりもはるかに安全に改修することができるでしょう。

これは蛇足ですが、業務では [GitLab](https://about.gitlab.com/) を使っており、上記コマンドを .gitlab-ci.yml に書いて、[スケジュール](https://gitlab-docs.creationline.com/ee/ci/pipelines/schedules.html)に登録しています。GitLab 便利ですね。

```..gitlab-ci.yml
fixNoReplyReporterIssues:
  rules:
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS'
      when: never
    - when: on_success
  script:
    - sbt fixNoreplyReporterIssues
```

# まとめ

sbt/scala と Java API を使ってサービスを操作する流れを説明しました。肝は次の通りです。

- project/plugin.sbt に必要な Java API の依存関係を記述する。
- build.sbt に日常業務タスクを記述する。
- project/**Utils.scala にサービスAPIと日常業務とのギャップを記述する。

maven central などにある Java API は、基本的にはビルドして使うイメージがあったかと思いますが、sbt の場合は `project/plugin.sbt` に書くことで、その依存ライブラリを build.sbt 内部で利用することができます。build.sbt 部分は明示的なコンパイルやパッケージ処理も不要なことを考えると、日常ツールとして使える気がしませんか？

ちなみに、maven central の Java API を明示的なコンパイル無しで使う方法として Groovy/Grape があります。しかし Groovy は動的型付言語であり、書き味は似せることができますが、保守面を考えると型の保護がある Scala の方が軍配が上がりそうな気がしています。
