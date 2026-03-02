https://qiita.com/kijuky/items/a22f42ebb465c43a40dc

---

[tavern](https://tavern.readthedocs.io/en/latest/) とは api テストツールです。いわゆる結合テストレベルのツールであり、[postman](https://www.postman.com/) をコンソールで実行できるものだと思ってもらえると良いかと思います。

今回、これを使いたかったのは [sbt-play-swagger を導入](https://qiita.com/__kijuky__/items/8182d5c5ec43c3ed87e8)して routes ファイルを修正したのですが、そのレグレッションテスト（退行確認テスト）を用意する必要がありました（業務だと多重チェック重要ですよね）。

どうでもいいですが、最初綴りを覚えてない頃は `tarven` ってよく間違えてました。ターバンではなくタバーンですね。英語で居酒屋という意味らしいです。ちなみに頭に巻くターバンの綴りは `turban` らしいです。

# インストール

```
$ pip install tavern
```

# 使い方

テストケースを表す yaml ファイルを用意します。ファイル名は `test_***.tavern.yaml` とし、 `***` の部分をテストシナリオ名とします。業務ではメソッド名とパス名を `_` で繋ぎ、パス名の `/` を `_` に一律置換したものを設定しました。レグレッションテストであまりこのへんの命名に脳味噌使いたくないですよね。

```test/tavern/test_get_hoge.tavern.yaml
test_name: get
includes:
  - !include common.yaml
stages:
  - name: get
    request:
      url: "{base_url}/hoge"
      method: GET
    response:
      status_code: 200
```

しれっと `!include common.yaml` を用意しています。これはQA環境のホスト名を環境変数から変更するための仕組みです。`HOST` 環境変数に接続ホスト名を設定しておくと、接続先をアドホックに変えることができます。

```test/tavern/common.yaml
name: common
description: common
variables:
  base_url: "http://{tavern.env_vars.HOST}"
```

あと、ちょっとハマったんですが、 yaml で値に `{}` を使う場合はダブルクォーテーションで囲うのは必須です。

準備が整ったら tavern-ci を実行します。

```
$ HOST='localhost:9000' tavern-ci test/tavern
```

こうすると `test/tavern` フォルダにある `test_*.tavern.yaml` ファイルが全て対象になります。フォルダは複数指定できます。サブプロジェクトにあるテストファイルを一括で指定する場合は `*/test/tavern` みたいに指定すると便利かもしれません。

# Tips

## パラメータがある場合

たとえば `param1=abc` というパラメータをつけてリクエストする場合はこんな感じになります。

```test/tavern/test_get_hoge.tavern.yaml
test_name: get
includes:
  - !include common.yaml
stages:
  - name: get
    request:
      url: "{base_url}/hoge"
      method: GET
      params:
        param1: abc
    response:
      status_code: 200
```

ちなみに、tavern は python で実行されるため、型情報が python に寄った結果、異なる出力を得ることがあります。例えば `param1: true` とした場合、trueが真偽値として扱われ、python の True になり、リクエストする際にはそれが文字列化されて `param1=True` としてリクエストされます。`true`と`True`を厳密に比較している場合は注意してください。

上記のことがあるため、基本的には params の値は全部ダブルクォーテーションで囲んでおいた方が無難です。

## POSTする場合

GET と同じで method を POST に変えるだけです。この場合は `{"param1":"abc"}` という JSON を body に設定した感じになります。

```test/tavern/test_post_hoge.tavern.yaml
test_name: get
includes:
  - !include common.yaml
stages:
  - name: get
    request:
      url: "{base_url}/hoge"
      method: POST
      params:
        param1: abc
    response:
      status_code: 200
```

## フォーム形式としてPOSTする場合

ログインフォームとかだと JSON 形式でポストしても困ります。フォーム形式の場合は `params` ではなく `data` を使います。

```test/tavern/test_post_hoge.tavern.yaml
test_name: get
includes:
  - !include common.yaml
stages:
  - name: get
    request:
      url: "{base_url}/hoge"
      method: POST
      data:
        param1: abc
    response:
      status_code: 200
```

## クッキーを使う場合

会員制のサイトのコンテンツを見る場合、セッションクッキーが設定されたリクエストでコンテンツを見る必要があると思います。stages には複数のリクエストを書けて、以前のステージで取得したクッキーは後続のステージで利用することができます。


```test/tavern/test_get_hoge.tavern.yaml
test_name: get
includes:
  - !include common.yaml
stages:
  - name: login
    request:
      url: "{base_url}/login"
      data:
        username: username
        password: password
      response:
        status_code: 303
        cookies: 
          - session_id
  - name: get
    request:
      url: "{base_url}/hoge"
      method: GET
    response:
      status_code: 200
```


## CI での利用

tavern は統合テストレベルなので、QA環境等にデプロイした後に実行できるようになっていると良いでしょう。

```.gitlab-ci.yml
stages:
  - deploy
  - integration

deploy:
  # 何らかデプロイ作業

tavern:
  stage: integration
  image: python:3.9-alpine
  variables:
    HOST: qa:9000
  script:
    - pip3 install tavern
    - tavern-ci test/tavern
```

## テスト結果をレポートとして出力する場合

[allure](https://docs.qameta.io/allure/) に対応しています。allure をインストールしていない場合は先にインストールする必要があります。

```
$ brew install allure
$ pip install allure-pytest
```

tavern-ci 実行時に引数に allure ログ出力先を指定してあげます。

```
$ HOST='localhost:9000' tavern-ci --allurelog=logs/tavern test/tavern
```

出力されたデータを手元で見る場合は serve します。

```
$ allure serve logs/tavern
```

アーティファクトとして保存したい場合は generate します。

```
$ allure generate --output report/tavern logs/tavern
```

allure のレポートはとてもリッチでいいですね。こちらのレポートにはリクエストとレスポンスの詳細な記録も残っているので、テストが失敗したときにこちらのレポートが残っていると、解析がしやすくなると思います。

# まとめ

- レグレッションテストを目的として tavern という api テストツールを使ってみました。結果、ちゃんとバグが見つかってよかったです（あぶねー）
- tavern はテストケースを yml で用意するので、Webフレームワークによらず知識が再利用できそうです。
