https://qiita.com/kijuky/items/136586048b4feceea04a

---

この挙動でバグらせてしまったので、供養のために書いておきます。

# Joda-Timeは書式文字列で指定した桁数以下でもパースできる

`yyyy/MM/dd HH:mm`という書式文字列で解析するとします。java.timeでは`2023/9/1 8:1`みたいな文字列は解析できません。`M`や`H`などの文字数は最低桁数として解釈されます。

```amm
@ import java.time._, java.time.format._ 
import java.time._, java.time.format._

@ LocalDateTime.parse("2023/09/01 08:01", DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm")) 
res6: LocalDateTime = 2023-09-01T08:01

@ LocalDateTime.parse("2023/9/1 8:1", DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm")) 
java.time.format.DateTimeParseException: Text '2023/9/1 8:1' could not be parsed at index 5
  java.time.format.DateTimeFormatter.parseResolved0(DateTimeFormatter.java:2052)
  java.time.format.DateTimeFormatter.parse(DateTimeFormatter.java:1954)
  java.time.LocalDateTime.parse(LocalDateTime.java:494)
  ammonite.$sess.cmd7$.<clinit>(cmd7.sc:1)
```

一方、Joda-Timeでは解析できます。

```amm
@ import $ivy.`joda-time:joda-time:2.12.5`  
import $ivy.$                            

@ import org.joda.time._, org.joda.time.format._ 
import org.joda.time._, org.joda.time.format._

@ DateTime.parse("2023/09/01 09:07", DateTimeFormat.forPattern("yyyy/MM/dd HH:mm")) 
res3: DateTime = 2023-09-01T09:07:00.000+09:00

@ DateTime.parse("2023/9/1 9:7", DateTimeFormat.forPattern("yyyy/MM/dd HH:mm")) 
res4: DateTime = 2023-09-01T09:07:00.000+09:00
```

# JavaDocを読む

比較しやすい様に英語で比べてみます。

## java.time

https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/format/DateTimeFormatter.html

> **Number**: If the count of letters is one, then the value is output using the minimum number of digits and without padding. Otherwise, the count of digits is used as the width of the output field, with the value zero-padded as necessary. The following pattern letters have constraints on the count of letters. Only one letter of 'c' and 'F' can be specified. Up to two letters of 'd', 'H', 'h', 'K', 'k', 'm', and 's' can be specified. Up to three letters of 'D' can be specified.

文字の数は最小桁数になる、と書いてあります。

## Joda-Time

https://www.joda.org/joda-time/apidocs/org/joda/time/format/DateTimeFormat.html

> **Number**: The minimum number of digits. Shorter numbers are zero-padded to this amount. When parsing, any number of digits are accepted.

解析する場合は、どんな数字でも受け付けると書いてあります。つまり、解析の際は文字の数は関係ないみたいですね。

# まとめ

JavaDocをよく読むと

- java.timeは、文字数は最小桁数として解釈
- Joda-Timeは、文字数は関係なく数字として解釈

することがわかります。これらを考慮して、Joda-Timeにて`yyyy/MM/dd HH:mm`として受け取っていた文字列をjava.timeで受け取る様にするためには`y/M/d H:m`という書式文字列にする必要があります。

```amm
@ import java.time._, java.time.format._ 
import java.time._, java.time.format._

@ LocalDateTime.parse("2023/09/01 08:01", DateTimeFormatter.ofPattern("y/M/d H:m")) 
res12: LocalDateTime = 2023-09-01T08:01

@ LocalDateTime.parse("2023/9/1 8:1", DateTimeFormatter.ofPattern("y/M/d H:m")) 
res13: LocalDateTime = 2023-09-01T08:01
```

ちなみに、この文字列であれば、Joda-Timeでも同様に解析できます。

```amm
@ import $ivy.`joda-time:joda-time:2.12.5`  
import $ivy.$                            

@ import org.joda.time._, org.joda.time.format._ 
import org.joda.time._, org.joda.time.format._

@ DateTime.parse("2023/9/01 9:7", DateTimeFormat.forPattern("y/M/d H:m")) 
res14: DateTime = 2023-09-01T09:07:00.000+09:00

@ DateTime.parse("2023/09/01 09:07", DateTimeFormat.forPattern("y/M/d H:m")) 
res15: DateTime = 2023-09-01T09:07:00.000+09:00
```
