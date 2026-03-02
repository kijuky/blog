https://qiita.com/kijuky/items/11a8cdc598953c81ea05

---

https://qiita.com/kijuky/items/374ebbd85d742f7221fe

こちらの続きです。前回までは手元で `sbt publishSigned` としないとセントラルリポジトリに登録できませんでしたが、タグをプッシュすることで GitHub Action によってセントラルリポジトリに登録できるようにしました。

ほとんど下記記事に従えば環境構築できますが、いくつかハマリポイントがあったので紹介していきます。

https://eed3si9n.com/ja/auto-publish-sbt-plugin-from-github-actions/

## プラグインの追加

```plugins.sbt
addSbtPlugin("com.geirsson" % "sbt-ci-release" % "1.5.7")
```

バージョンが低いとうまく動かないので、必ず最新バージョンを使用してください。

## sbt の編集

まず、前回作った publish.sbt は削除します。続けて、 build.sbt から下記を削除します。

- organization
- version
- publishTo
- publishMavenStyle
- credentials

次に下記を追記します。

```build.sbt
inThisBuild(Seq(
  organization := "$organization$",
  homepage := Some(url("$homepage$")),
  licenses := Seq("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0")),
  developers := List(
    Developer(
      "$GitHubAccount$",
      "$name$",
      "$email$",
      url("$GitHubAccountPage$")
    )
  ),
  versionScheme := Some("early-semver")
))

sonatypeCredentialHost := "s01.oss.sonatype.org"
```

- \$organization$: `groupId`
- \$homepage$: ホームページ
- \$GitHubAccount$: アカウント名
- \$name$: 本名
- \$email$: メール
- \$GitHubAccountPage$: アカウントページ
- sonatypeCredentialHost: sonatype のホスト名。最近アカウントを作ると **s01.**oss.sonatype.org になるので、必ず設定する。 oss.sonatype.org の場合は省略可

## gpg2 用の設定

署名に使う gpg のバージョンが 2.2 以上だと、秘密鍵を自前でデコードする必要があります。下記ファイルを用意します

```.github/decodekey.sh
#!/bin/bash
 
echo $PGP_SECRET | base64 --decode | gpg  --batch --import
```

実行権限をつけます。

```shell
$ chmod +x .github/decodekey.sh
```

## GitHubAction の追加

下記ファイルを用意します。

```.github/workflows/release.yml
name: Release
on:
  push:
    tags: ["*"]
jobs:
  publish:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2.3.4
        with:
          fetch-depth: 0
      - uses: olafurpg/setup-scala@v10
      - run: .github/decodekey.sh && sbt ci-release
        env:
          PGP_PASSPHRASE: ${{ secrets.PGP_PASSPHRASE }}
          PGP_SECRET: ${{ secrets.PGP_SECRET }}
          SONATYPE_PASSWORD: ${{ secrets.SONATYPE_PASSWORD }}
          SONATYPE_USERNAME: ${{ secrets.SONATYPE_USERNAME }}
```

一通りのファイルが揃ったら、コミットしておきます。

## 環境変数の設定

前回で gpg キーは作りましたが、自動リリース用に別に gpg キーを作っておきます。

```shell
$ gpg --gen-key
```

- パスワードは大文字小文字記号を含めておいてください（ある程度複雑でないとキー生成に失敗します）
- ここで本名は何でもいいです。元記事では `<<製品名>> bot` としていました。
- Eメールは連絡のつくメールアドレスにしておきましょう。

出てきたキーの uuid を http://keyserver.ubuntu.com:11371/ に送信します。

```
...
pub   rsa4096 2018-08-22 [SC]
      1234517530FB96F147C6A146A326F592D39AAAAA
uid           [ultimate] your name <you@example.com>
sub   rsa4096 2018-08-22 [E]
$ gpg --keyserver hkp://keyserver.ubuntu.com --send-keys 1234517530FB96F147C6A146A326F592D39AAAAA
```

次に GitHub の https://github.com/<\<account>>/<\<project>>/settings/secrets/actions にアクセスし、GitHub Action 時に利用する環境変数を設定します。

| 環境変数名 | 内容 |
|:-:|:-:|
| PGP_PASSPHRASE | 作った gpg キーのパスフレーズ |
| PGP_SECRET | 作った gpg キーのBASE64化した秘密鍵(後述) |
| SONATYPE_PASSWORD | sonatype のパスワード |
| SONATYPE_USERNAME | sonatype のユーザー名 |

秘密鍵は次のように base64 化します。ここで 1234... は実際に作ったキーのものに差し替えます。エクスポートする際にパスフレーズの入力が要求されます。

```shell
gpg --armor --export-secret-keys 1234517530FB96F147C6A146A326F592D39AAAAA | base64 | pbcopy
```

## パブリッシュする

お疲れさまでした。ここまで来たらあとは commit に tag をつけてプッシュするだけです。注意点として、タグは `v` から始まり、セマンティックバージョニングの形式で記述する必要があります。

```shell
git tag -a v0.1.0 -m "v0.1.0"
git push origin v0.1.0
```

# 参考

https://github.com/olafurpg/sbt-ci-release

https://github.com/xerial/sbt-sonatype

https://eed3si9n.com/ja/auto-publish-sbt-plugin-from-github-actions/
