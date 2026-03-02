https://qiita.com/kijuky/items/534dc5de633754072b7c

---

最近、業務でPlayの開発が増えて段々とPlayに関しての知識が増えてきました。せっかくなのでクラウドサービスにデプロイしてみたいと思います。今回は無料で利用できると有名な Heroku を使ってみたいと思います。

# Heroku とは

Webアプリケーションの実行環境を提供しているサービスです。このサービスに対してアプリをデプロイすることで、他の人がアプリを利用できるようになります。すごいですね。

言語別のチュートリアルもあって、とても分かりやすかったです。この記事では以下の内容のダイジェストになります。

https://devcenter.heroku.com/articles/getting-started-with-scala

## [CLIツールのインストール](https://devcenter.heroku.com/articles/getting-started-with-scala#set-up)

まずはデプロイするための heroku コマンドをインストールします。

```console
% brew install heroku/brew/heroku
...（省略）
% heroku --version
 ›   Warning: Our terms of service have changed: 
 ›   https://dashboard.heroku.com/terms-of-service
heroku/7.62.0 darwin-x64 node-v14.19.0
```

## [デプロイ](https://devcenter.heroku.com/articles/getting-started-with-scala#prepare-the-app)

サクッとサンプルアプリをクローンします。

```console
% git clone https://github.com/heroku/scala-getting-started.git
% cd scala-getting-started
```

このサンプルアプリのプッシュ先を heroku クラウド上に作成します。

```console
% heroku create
% git push heroku main
```

`heroku create` でリモートに `heroku` が作成されて、`git push` 先に `heroku` を設定する、と。面白いですね。

動作確認は

```console
% heroku open
```

## [ログの確認](https://devcenter.heroku.com/articles/getting-started-with-scala#view-logs)

```console
% heroku logs --tail
```

## [起動オプション](https://devcenter.heroku.com/articles/getting-started-with-scala#define-a-procfile)

[Procfile](https://devcenter.heroku.com/articles/procfile) と呼ばれるファイルに、そのアプリの起動オプションを追加します。Playでは起動オプションでconfを切り替えることが多いと思うので、これはチェックしておくと良いですね。

```:Procfile
web: target/universal/stage/bin/play-getting-started -Dhttp.port=${PORT}
console: target/universal/stage/bin/play-getting-started -main scala.tools.nsc.MainGenericRunner -usejavacp
```

## [コンテナ(dyno)](https://devcenter.heroku.com/articles/getting-started-with-scala#scale-the-app)

上記のオプションで起動されたアプリをコンテナ(dyno)で実行します。実行状態を確認してみます。

```console
% heroku ps
```

クラウドサービスは、負荷に応じてコンテナをスケールさせることができます。ただ、heroku は無料枠ではスケールできませんし、そもそも Play のようなオールインワンWebフレームワークをスケールさせるのはリソース面であまり良いものではありませんね。

例えば、無料枠でもアプリを複数持っていて、他のアプリにコンテナを明け渡したい、みたいな場合、一旦アプリに割り振るコンテナを0に割り振ることができます。

```console
% heroku ps:scale web=0
```

`web` に割り当てるコンテナを `0` にする、と言う意味です。復活させる場合は

```console
% heroku ps:scale web=1
```

## [ローカルでの開発](https://devcenter.heroku.com/articles/getting-started-with-scala#declare-app-dependencies)

まずは `system.properties` で Java のバージョンを確認しておきます。

```system.properties
java.runtime.version=1.8
```

ビルドします。注意として現在のサンプルアプリが sbt 0.13 系を使っており、依存関係が ivy を使っている都合上、めちゃくちゃ時間がかかります。一通りビルドできたら、sbt は 1 系にあげておくのをお勧めします。

```console
% sbt compile stage
```

ローカルデプロイします。

```console
% heroku local web
```

## [構成変数](https://devcenter.heroku.com/articles/getting-started-with-scala#define-config-vars)

いわゆるシークレットの管理です。ここではチュートリアルに倣って `ENERGY="20 GeV"` という変数を定義してみます。リモートとローカルで設定方法が違うので注意です。

### リモート

`heroku config` を使います。

```console
% heroku config:set ENERGY="20 GeV"
```

### ローカル

`.env` ファイルを変更します。

```:.env
ENERGY=20 GeV
```

## [データベース](https://devcenter.heroku.com/articles/getting-started-with-scala#use-a-database)

heroku には容量制限があるとはいえ無料でそのアプリ専用のデータベースが付いてきます。ただし、このデータベースはコンテナと寿命が一緒で、つまり無料枠だと30時間アクセスがないとなくなってしまいます。本当に一時的なデータベースなので、永続化が必要なデータはちゃんとしたデータベースを用意しましょう。

まずはそのhobby用のデータベースのスペックを確認します。

```console
% heroku pg
```

実際にアプリで有効にする場合は application.conf のコメントアウトを外します。

```hocon:application.conf
db.default.driver=org.postgresql.Driver
db.default.url=${?DATABASE_URL}
```

db のコンソールに入るには次のようにします。

```console
% heroku pg:psql
```

# まとめ

これだけの機能が揃っていて、無料で始められるのはすごいですね。ちょっと遊んでみようと思います。
