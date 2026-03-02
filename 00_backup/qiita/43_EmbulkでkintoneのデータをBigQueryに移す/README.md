https://qiita.com/kijuky/items/36682f85bc1118859957

---

タイトル通りです。

# Embulk とは

https://www.embulk.org/

あるデータベースにあるデータを、別のデータベースにコピーできるツール（雑　プラグインを導入することで入出力先のデータベースとしていろいろなものを取れる。今回は入力先に kintone を、出力先に BigQuery を使いました。

# kintone とは

https://kintone.cybozu.co.jp/

GUI データベースです（雑　GUIを備えているので非プログラマでもデータ管理ができます。

# BigQuery とは

https://cloud.google.com/bigquery?hl=ja

データベースです（雑　名前に「クエリー」とあるように、大量のデータに対してSQLを使った抽出を高速に処理できます。

# EmbulkでkintoneのデータをBigQueryに移す

BigQuery には大量のデータから高速に抽出処理ができるので、社内のいろいろなデータをとりあえず BigQuery に置いておく、あると思います。同じように kintone のデータも BigQuery 上にあれば、抽出処理が BigQuery で完結します。抽出結果は redash とか tablue とかを使ってグラフィカルに表示できる先が既にあるので、社内のデータの「見える化」が捗る、ってやつです。

## Embulkでkintoneのデータを取り出す

まずは kintone のデータを Embulk を使って持ってきてみましょう。それには [embulk-input-plugin](https://github.com/trocco-io/embulk-input-kintone) を使います。

https://qiita.com/giwa/items/f571ed00b98c3d357b29

まずは対象の kintone のアプリケーションの [APIトークン](https://jp.cybozu.help/k/ja/user/app_settings/api_token.html) と [フィールドコード](https://jp.cybozu.help/k/ja/user/app_settings/form/autocalc/fieldcode.html) を設定します。ここで、**フィールドコードはそのまま BigQuery でのカラム名になるため、英数字で設定**しておきましょう。特に日本語はだめです。BigQuery に移さないフィールドに関しては無視されるので、全てのフィールドのフィールドコードを見直す必要はありません。

[設定したら「アプリの更新」を行うのを忘れずに](https://kintoneapp.zendesk.com/hc/ja/articles/4411455820441)。

とりあえずサクッと取り出した結果を見たい場合は上記の Qiita 記事にもあるような `config.yml` を用意して `embulk` を叩いてみましょう。

```config.yml
in:
  type: kintone
  domain: <<ここにdomainをいれる>>
  token: <<ここにAPIトークンを入れる>>
  app_id: <<ここにアプリIDを入れる>>
  fields:
    - {name: $id, type: long}
    - {name: <<ここにフィールドコードをいれる>>, type: string}
    ...(以降たくさん）
filters:
  - type: rename
    columns:
      $id: id
out: 
  type: stdout
```

```console
$ embulk preview config.yml
```

`$id` は、全てのアプリに共通してある属性の1つで、row indexを表します。ここで注意として、BigQueryでは `$` の文字がカラム名に使えないので、filters で置換する必要があります。そのため、既にこれ以外で連携していて、フィールドコードを変えられない場合は、この filters で処理してしまうのも1手です（個人的にはそういう読み替えがいろんな連携先で行われると共通言語として利用しづらくなるので推奨はしません）。

`type` に関しては `string` で取っておくのが無難です。特に日付データは [embulk-input-plugin によって UTC に変換される](https://github.com/trocco-io/embulk-input-kintone/blob/master/src/main/java/org/embulk/input/kintone/KintoneInputColumnVisitor.java#L91)ため、文字列として取ってきた日付データをBigQueryの関数で読み替えた方が良さそうです。

## EmbulkでBigQueryにデータを移す

上記の out を bigquery に変更します。

```config.yml
in:
  ...上と同じ...
out:
  type: bigquery
  table: <<テーブル名>>
  auth_method: json_key
  auto_create_table: true
  compression: GZIP
  dataset: <<データセット名>>
  file_ext: .jsonl.gz
  formatter:
    type: jsonl
  json_keyfile: <<認証のキーファイル名>>
  mode: replace
  project: <<プロジェクト名>>
  schema_file: schema.json
  source_format: NEWLINE_DELIMITED_JSON
```

```schema.json
[
  {
    "description": "id",
    "type": "INT64",
    "name": "id",
    "mode": "REQUIRED"
  },
  {
    "description": "<<フィールドの説明>>",
    "type": "STRING",
    "name": "<<フィールドコード>>",
    "mode": "NULLABLE"
  }
]
```

スキーマの `name` が、filter 後のフィールドコードと一致している必要があります。`$id` は filters で `id` になったので、`name: "id"` となります。`mode` は、基本的には `NULLABLE` が無難でしょう。kintone は必須フィールドが空のデータも存在できるので、kintone 上の「必須」はあまり役に立ちません...

# まとめ

そこかしこにちょっとした落とし穴があったので、Qiita にまとめてみました。実際、私はそれらにハマりまくって識者の時間をまぁまぁ溶かしたので、同じことしたい人がまた引っかからないように、この駄文を残しておきます。
