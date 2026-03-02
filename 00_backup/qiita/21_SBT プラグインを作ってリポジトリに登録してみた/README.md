https://qiita.com/kijuky/items/374ebbd85d742f7221fe

---

Scala 開発に欠かせないビルドツールである [sbt](https://www.scala-sbt.org/1.x/docs/ja/Getting-Started.html)。今回はこのプラグインを開発してセントラルリポジトリ(sonatype)に登録してみたので、その方法と詰まったところを紹介します。

# 背景

ギョームで [yamory](https://yamory.io/) という脆弱性管理ツールを使っているのですが、こちらの [sbt プロジェクトのスキャン方法がワンライナーで頑張る](https://yamory.io/docs/command-scan-sbt/#gsc.tab=0)感じでした。手元で実行する場合はシェルスクリプトになっていると便利なのでシェルを書いていたのですが、シェルスクリプトの実行ディレクトリを取得しないと sbt が管理するプロジェクトが見つからないので実行結果が不安定になりました。そこで、そこに sbt というビルドツールがあるのになぜ前時代的なシェルスクリプトで手こずる必要があるのだろう？と疑問に思い、全て sbt で完結したくなりました。

# sbt 独自タスクの作成

https://www.casleyconsulting.co.jp/blog/engineer/897/

そもそも sbt で独自のタスクを作ったことがなかったので、yamory のワンライナーを実行する、みたいな自由度があるのかすらわかりませんでした。なのでまずはリファレンスを見ました。

https://www.scala-sbt.org/1.x/docs/ja/Custom-Settings.html

ものすごくざっくり説明すると、キーとセッティングという２つを定義する必要があり、キーがインターフェースでセッティングが実装、みたいなイメージです。普段 `sbt compile` みたいに実行していると思いますが、 `compile` というのがキーで、 `compile` によって実際に行われる動作がセッティング、といったイメージです。例えば

```build.sbt
val hello = taskKey[Unit]("hello task") // これがキー

lazy val root = (project in file("."))
  .settings(
    ...
    hello := { // これがセッティング
      println("hello")
    }
  )
```

これは `hello` というキーを定義し、 root プロジェクトに `hello` と言うキーは `println("hello")` を実行する、というセッティングを設定した、ということ。これで `sbt hello` を実行すると `hello` が実行されます。

つまり、このセッティングに先程のワンライナーを書けばいいわけだな。：kanzennnirikaishita:

# sbt プラグインの作成

sbt プラグインとは、事前にキーとセッティングを定義し、それを組み込んだプロジェクトで独自のタスクが実行できるようにしたものです。ここで sbt の面白いところは sbt プラグイン自体も sbt プロジェクトである、というところ。なので、普通の sbt/scala プロジェクトと同様に開発が可能です。

[sbt プラグインには雛形がある](https://github.com/sbt/sbt-autoplugin.g8)ので、まずはそちらでサクッとプロジェクトを作ります。

```shell
$ sbt new sbt/sbt-autoplugin.g8
```

（もしかしたら、これ古いかも？）

作ったプロジェクトで、実際のタスクを定義するのは `src/main/scala/$package$/XxxPlugin.scala` というファイル。ここに先程のキーとセッティングを追加します。ただし、キーは `autoImport` オブジェクトに定義し、セッティングは `projectSettings` の Seq に追加します。

```src/main/scala/$package$/XxxPlugin.scala
...
class XxxPlugin extends AutoPlugin {
...
  object autoImport {
    val hello = taskKey[Unit]("A task that is print 'hello'.")
  }
...
  override lazy val projectSettings = Seq(
    hello := helloTask.value
  )

  lazy val helloTask = Def.task {
    println("hello")
  }
}
```

定義する場所が変わりましたが、この `helloTask` に独自タスクを定義することで、プラグインが実装できます。簡単だね！

実際に書いたものがこちら

https://github.com/kijuky/yamory-sbt-plugin/blob/master/src/main/scala/io/github/kijuky/sbt/plugins/yamory/YamoryPlugin.scala

手間取ったのが、取得した文字列を外部プロセスに標準入力として与える方法がなかったところ。下記を参考に、一時ファイルを作ってそれをリダイレクトに設定しました。

https://ameblo.jp/zuruzuru4/entry-12101545619.html

## 動作確認

作ったタスクがちゃんと動作するか、ローカルにパブリッシュして適当な sbt プロジェクトに読み込ませてみましょう。ローカルにパブリッシュするには下記を実行します。

```shell
$ sbt publishLocal
```

その後、適当な sbt プロジェクトを作成し

```shell
$ sbt new sbt/scala-seed.g8 --name="sample"
```

パブリッシュしたプラグインを読み込ませます。

```shell
$ cd sample 
$ cat <<'EOF'> project/plugins.sbt
addSbtPlugin("$package$" % "hello-plugin" % "0.1.0-SNAPSHOT")
EOF
$ sbt hello
```

# sonatype リポジトリへの登録

https://www.scala-sbt.org/1.x/docs/Using-Sonatype.html

せっかく作ったプラグインをみんなに使ってもらうためにはセントラルリポジトリというところにプラグインを登録する必要があります。雛形の sbt-autoplugin には [bintray](https://bintray.com/) と呼ばれるリポジトリに登録する設定がありますが、[このリポジトリは snoatype に移行したようです](https://eed3si9n.com/ja/bintray-to-jfrog-artifactory-migration-status-and-sbt-1.5.1)。今回は [sonatype](https://central.sonatype.org/) にプラグインを登録してみました。

## GPGキーの作成

プラグイン作者が本人であることの確認のために GPG と呼ばれるキーを作成/登録する必要があります。

```shell
$ brew install gpg # gpg が入っていない場合
$ gpg --gen-key
# ここで本名やらメールアドレスやらを入力する
```

パスフレーズの入力が促されますが、パスフレーズは十分に複雑でないと入力が拒否されるようです。ただし、拒否されたあとも「パスフレーズを入力してください」としか出ないので、注意が必要です。だいたい大文字小文字記号を使っておくと良いようです。入力後は TAB キーでOKをハイライトし、Enter を押下します。

するとキーが作成されるのでサーバーに登録します。 `--send-keys` には表示された `uid` をコピペします。

```shell
...
pub   rsa4096 2018-08-22 [SC]
      1234517530FB96F147C6A146A326F592D39AAAAA
uid           [ultimate] your name <you@example.com>
sub   rsa4096 2018-08-22 [E]
$ gpg --keyserver hkp://pool.sks-keyservers.net --send-keys 1234517530FB96F147C6A146A326F592D39AAAAA
```

作成した gpg キーを sbt プロジェクトで利用できるように、下記をグローバルに設定します。

```~/.sbt/1.0/plugins/gpg.sbt
addSbtPlugin("com.github.sbt" % "sbt-pgp" % "2.1.2")
```

## Sonatype への登録

https://central.sonatype.org/publish/publish-guide/

次に、パブリッシュ先の sonatype に自身の情報を登録します。下記動画にあるように、登録には[JIRAのアカウントを作成](https://issues.sonatype.org/secure/Signup!default.jspa)後、[JIRA の issue を作ります](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134)。

https://youtu.be/0gyF17kWMLg

作成した JIRA の issue がこちら。

https://issues.sonatype.org/browse/OSSRH-69841

リポジトリの許可には2営業日くらいかかるみたいに書いてありますが、実際には作ってから1時間もしないうちにリポジトリの許可がおりました。いくつか注意点があるので紹介します。

### groupId は `com.github.アカウント名` ではなく `io.github.アカウント名`

https://central.sonatype.org/changelog/#2021-04-01-comgithub-is-not-supported-anymore-as-a-valid-coordinate

新規でGithubに紐付いた個人アカウントを作る場合、 `com.github` はサポートされなくなったようです（すでに `com.github` を持っている人はそのまま使えます）。新規の場合は `io.github.アカウント名` を使います。

### JIRAチケット番号の空リポジトリを作る

「ほんとうにそのGithubアカウントの所有者か？」を確認するために、JIRAチケット番号の名前のPublicな空リポジトリを作る必要があります。へーという感じ。

一連の確認が終わると「ここにパブリッシュせーよ」とコメントをもらいました。ここまでくれば実際にパブリッシュできます。

> io.github.kijuky has been prepared, now user(s) kijuky can:
Publish snapshot and release artifacts to s01.oss.sonatype.org

### パブリッシュ先のホスト名に注意

リンク先の説明に書いてあるサンプルコードとはホスト名が違っていたので、若干手間取りました。今回私が許可されたパブリッシュ先は [**s01.**oss.sonatype.org](https://s01.oss.sonatype.org/#welcome) でした。認証情報にそれらの情報を反映させなければいけないので、注意が必要です。

## sbt にパブリッシュ情報を追加

ここまででパブリッシュ先ができたので、実際に今回作った sbt プラグインをパブリッシュしてみました。まず sonatype の認証情報をグローバルに設定します。

```~/.sbt/1.0/sonatype.sbt
credentials += Credentials(Path.userHome / ".sbt" / "sonatype_credentials")
```

上記で参照しているファイルを作成します。

```~/.sbt/sonatype_credentials
realm=Sonatype Nexus Repository Manager
host=s01.oss.sonatype.org
user=<your username>
password=<your password>
```

- host は **s01.**oss.sonatype.org
- user は、JIRAに登録したユーザー名。**メールアドレスではないことに注意**
- password は、JIRAに登録したパスワード

この辺の認証情報を間違えると、403 だったり（hostを間違えた場合）、ループバックしたり（userを間違えた場合）するので注意が必要です。

https://githubmemory.com/repo/sbt/sbt-pgp/issues/182

次に、プロジェクトルートで publish.sbt を作成して、以下をコピペします。 `organization` は build.sbt にも定義されていることがあるので、どちらか一方にだけ定義するようにします（私は publish.sbt の方を残しました）。その他の変更箇所を \$~~$ で紹介します。

```publish.sbt
ThisBuild / organization := "$organization$"
ThisBuild / organizationName := "$organizationName$"
ThisBuild / organizationHomepage := Some(url("$projectHomepage$"))

ThisBuild / scmInfo := Some(
  ScmInfo(
    url("https://github.com/$githubAccount$/$githubProjectName$"),
    "scm:git@github.com:$githubAccount$/$githubProjectName$.git"
  )
)
ThisBuild / developers := List(
  Developer(
    id    = "$githubAccount$",
    name  = "$name$",
    email = "$email$",
    url   = url("$projectHomepage$")
  )
)

ThisBuild / description := "$description$"
ThisBuild / licenses := List("Apache 2" -> new URL("http://www.apache.org/licenses/LICENSE-2.0.txt"))
ThisBuild / homepage := Some(url("$projectHomepage$"))

// Remove all additional repository other than Maven Central from POM
ThisBuild / pomIncludeRepository := { _ => false }
ThisBuild / publishTo := {
  val nexus = "https://$sonatypeHost$/"
  if (isSnapshot.value) Some("snapshots" at nexus + "content/repositories/snapshots")
  else Some("releases" at nexus + "service/local/staging/deploy/maven2")
}
ThisBuild / publishMavenStyle := true
```

- \$organization$ `groupId`
- \$organizationName$ `artifactId`
- \$projectHomepage$ プロジェクトのホームページ。
- \$githubAccount$ Githubのアカウント名
- \$githubProjectName$ Githubのプロジェクト名
- \$name$ 本名
- \$email$ Eメールアドレス
- \$description$ このプロジェクトの説明
- \$sonatypeHost$ パブリッシュ先のホスト名。 **s01.**oss.sonatype.org

最後に、Intellij IDEA の console でやっている人は[下記のおまじないを入力します](https://github.com/keybase/keybase-issues/issues/2798)。

```shell:IntelliJ_IDEA_console
$ export GPG_TTY=$(tty)
```

ここまで入力が終わったら、下記コマンドでパブリッシュできます（初回だけ gpg キーのパスフレーズの入力が必要です）。

```
$ sbt publishSigned
```

実際にパブリッシュしたものが以下になります。

https://s01.oss.sonatype.org/service/local/repositories/snapshots/content/io/github/kijuky/yamory-sbt-plugin_2.12_1.0/1.0.0-SNAPSHOT/yamory-sbt-plugin-1.0.0-SNAPSHOT.pom

# まとめ

- [sbt プラグインを作った](https://github.com/kijuky/yamory-sbt-plugin)
- 作ったプラグインを sonatype に登録できた

# 今後

- [Github Action を使って登録を自動化できる](https://eed3si9n.com/ja/auto-publish-sbt-plugin-from-github-actions)ようなので、やってみる。

## 追記

つづき

https://qiita.com/__kijuky__/items/11a8cdc598953c81ea05

# 参考

https://eed3si9n.com/ja/bintray-to-jfrog-artifactory-migration-status-and-sbt-1.5.1
