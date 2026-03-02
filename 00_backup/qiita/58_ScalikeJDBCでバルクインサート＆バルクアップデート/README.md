https://qiita.com/kijuky/items/261477accfa8ee52eb6f

---

SQLでたくさんのデータをインサートしたりアップデートしたりすることを探していたら、意外とニッチな要求らしく、ググるにしても難しく、かつそれをScalikeJDBCでやるにはどうしたらいいんだろう？と躓いたので、メモとして残しておきます。

# バルクインサート、バルクアップデートとは？

手でSQL書くには多すぎる、例えば20件を超えるようなデータを一気にインサートしたりアップデートしたりすることを指します。後のScalikeJDBCでSQL Interpolationで説明するので、実際のSQL例を提示して説明してみます。

## バルクインサート

SQLのインサートの構文はもともとバルクインサートできるようになっています。

```sql
insert into hoge(id, fuga, piyo)
  values (1, 'fuga1', 'piyo1'),
         (2, 'fuga2', 'piyo2'),
         (3, 'fuga3', 'piyo3'),
         (4, 'fuga4', 'piyo4'),
         ...省略
         (20, 'fuga20', 'piyo20')
```

## バルクアップデート

バルクアップデートはちょっと工夫が必要になります。

```sql
update hoge as h
  set fuga = updated.fuga
      piyo = updated.piyo
  from (values
      (1, 'FUGA1', 'PIYO1'),
      (2, 'FUGA2', 'PIYO2'),
      ...省略
      (20, 'FUGA20', 'PIYO20')
    ) as updated(id, fuga, piyo)
  where h.id = updated.id
```

fromに続けてアップデートするデータをvaluesで指定する。また、setに指定するフィールド名は`h.fuga = updated.fuga`とかするとすぐ怒られるので注意。

https://qiita.com/yuuuuukou/items/d7723f45e83deb164d68#%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B32-%E5%90%84%E5%80%A4%E3%82%92%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%81%A8%E3%81%97%E3%81%A6%E6%B8%A1%E3%81%99

https://qiita.com/kt215prg/items/55de78e057c99edcecab

# ScalikeJDBCを使う

https://github.com/scalikejdbc/scalikejdbc-cookbook/blob/master/ja/00_intro.md

ScalikeJDBCではデータをsqlsで作って、それをsqls.csvで挿入するのが肝です。上記クックブックでもSQLInterpolationが推奨されているので、例はSQLInterpolationを利用します。

## バルクインサート

```scala
val values: Seq[Hoge] = ... // インサートするデータ
val sqlsValues = values.map { s =>
  sqls"""
    |(
    |  ${s.fuga},
    |  ${s.piyo}
    |)""".stripMargin
}

// bulk insert
sql"""
  insert into hoge(fuga, piyo)
    values ${sqls.csv(sqlsValues: _*)}
  """.update().apply()
```

## バルクアップデート

```scala
val values: Seq[Hoge] = ... // アップデートするデータ
val sqlsValues = values.map { s =>
  sqls"""
    |(
    |  ${s.id},
    |  ${s.fuga},
    |  ${s.piyo}
    |)""".stripMargin
}

// bulk update
sql"""
  update hoge as h set fuga = updated.fuga, piyo = updated.piyo
    from (values ${sqls.csv(sqlsValues: _*)}) as updated(id, fuga, piyo)
    where h.id = updated.id
  """.update().apply()
```

# SQLInterpolationについてのTips

ScalikeJDBCのSQLInterpolationで`${...}`は常に`?`に置換されます。そのため、それを見越したコードを書く必要があります。

## 値の生成が複雑な場合

例えば複数の値を使う場合は一時変数で作っておきます。

```scala
val sqlsValues = values.map { s =>
  val fuga = s"fuga-${s.fuga}"
  sqls"""
    |(
    |  ${fuga},
    |  ${s.piyo}
    |)""".stripMargin
}
```

## timestamp型を設定する場合

java.time.LocalDateTimeをtimestamp型に設定する場合、postgresql/h2互換の設定方法は`::timestamp`サフィックスを追加します。

```
val sqlsValues = values.map { s =>
  sqls"""
    |(
    |  ${s.fuga},
    |  ${s.piyo},
    |  ${s.updated_at}::timestamp
    |)""".stripMargin
```
