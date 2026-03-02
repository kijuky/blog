https://qiita.com/kijuky/items/2bfa85f135d42d5558a4

---

ギョームで使って便利だなーと思ったので備忘録。

Kotlinは引数の初期化に引数が使えます。

```kotlin
data class A(
  val a: String,
  val aa: String = a.repeat(2),
)
```

```
Welcome to Kotlin version 1.5.10 (JRE 15.0.2+7)
Type :help for help, :quit for quit
>>> data class A(val a: String, val aa: String = a.repeat(2))
>>> println(A(a = "hello "))
A(a=hello , aa=hello hello )
```

面白いね！

# 役立ったところ

（一部脚色してます）性別や職種に応じてデフォルトの敬称(Honorific)を設定するのに使いました。

```kotlin
data class User(
  val name: String = "",
  val sex: Sex = Sex(),
  val occupation: Occupation = Occupation(),
  val honorific: String = honorific(sex, occupation),
) {
  companion object {
    private fun honorific(
      sex: Sex,
      occupation: Occupation,
    ): String {
      return when {
        (なんか条件１) -> "さん"
        （なんか条件２） -> "くん"
      }
    }
  }
}
```
