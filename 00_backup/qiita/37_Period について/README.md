https://qiita.com/kijuky/items/0dd816314ef13d27fe5f

---

ハマったのでメモ。

# 結論

閏日(2/29)から 2/28 までの Period が返す年数は、Joda Time と Java Time で結果が異なる。

# 環境

| ソフトウェア | バージョン |
|:-:|:-:|
| Joda Time | 2.10.8 |
| Java | 17.0.2 |

# 動作確認

1992-02-29 ~ 2022-02-28 までの年数を Period で取得してみる。

## Joda Time

```scala
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import $ivy.`joda-time:joda-time:2.10.8` 
import $ivy.$                           

@ import org.joda.time._ 
import org.joda.time._

@ new Period(new LocalDate(1992,2,29), new LocalDate(2022,2,28)).getYears 
res2: Int = 30
```

Joda Time では 30 年と出る。

## Java Time

```scala
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import java.time._ 
import java.time._

@ Period.between(LocalDate.of(1992,2,29), LocalDate.of(2022,2,28)).getYears 
res2: Int = 29
```

Java Time では 29 年と出る。

# 追記

1992-02-28 ~ 2022-02-27 は Joda Time でも Java Time でも 29 年と出る。

## Joda Time

```scala
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import $ivy.`joda-time:joda-time:2.10.8`  
import $ivy.$                            

@ import org.joda.time._ 
import org.joda.time._

@ new Period(new LocalDate(1992,2,28), new LocalDate(2022,2,27)).getYears 
res3: Int = 29
```

## Java Time

```scala
Welcome to the Ammonite Repl 2.5.2 (Scala 2.13.8 Java 17.0.2)
@ import java.time._ 
import java.time._

@ Period.between(LocalDate.of(1992,2,28), LocalDate.of(2022,2,27)).getYears 
res1: Int = 29
```

# 補足

- 上記は年齢を算出するロジックの単体テストで発覚した。
- 民法では、年齢は前日に +1 されるため、誕生日が 1992-02-29 の人は 2022-02-28 の時に 30 歳である。
    - 誕生日当日に +1 してしまうと、誕生時点で +1 されるため、0 歳が表現できなくなる。（から前日に +1 されるのかな？）
    - でもその場合、2022-02-28 ではまだ 29 歳のはず（24時を超えないと +1 しないため、実質 2022-03-01 で 30 歳のはず）なのに、民法では 2022-02-28 でもう 30 歳と扱うらしい。謎い...
