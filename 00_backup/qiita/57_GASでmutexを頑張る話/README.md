https://qiita.com/kijuky/items/465dca1775c511c226f5

---

:::note alert
この記事はポエムです。実際にロック実装を行う場合は LockService を使ってください。

https://qiita.com/kyamadahoge/items/f5d3fafb2eea97af42fe
:::


Google Apps Script（GAS）を用いてWeb APIと連携する際、通常はアクセストークンを利用して通信が行われます。一部のAPIではアクセストークンの有効期限が設定されており、新しいアクセストークンを取得する必要があります。アクセストークンを都度取得することも可能ですが、既に取得済みのトークンを再利用することで余分なリクエストを抑えることができ、経済的です。これにより、異なるスレッドで同じトークンを共有することが求められます。GASではPropertiesを利用することで、簡単に共有リソースが実現できます。

本記事では、Propertiesを使ってトークンを共有する仕組みと、mutexの実装方法を紹介します。

# mutex

mutex とは相互排他(mutual exclusion)の略であり、特定の共有リソースを触るのは1スレッドのみに限定する概念です。例えば、トイレの個室などは mutex の例ですね。ある人がトイレを使用中であれば、別の人がそのトイレを使用したい場合は、トイレが空くまで待つ必要があります[^1]。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/76fc75fa-5261-aa8e-983c-da900f507f04.png)
https://www.irasutoya.com/2018/07/blog-post_55.html

```plantuml
Actor A
Boundary トイレ
Actor B

A -> トイレ : 使用中
activate トイレ
B --> トイレ : Aが終わるまで待つ
トイレ --> A : スッキリ
deactivate トイレ
B -> トイレ : 使用中
activate トイレ
トイレ --> B : スッキリ
deactivate トイレ
```

mutex とは、この例で言うところのトイレの使用ルールを表す概念のようなものです。

# 前提

今回は[J-Quants API](https://jpx.gitbook.io/j-quants-ja/outline/getstarted)を例に話を進めます。こちらのAPIは、有効なAPIを叩くためにIDトークンが必要で、IDトークンを発行するにはリフレッシュトークンが必要になります。リフレッシュトークンは自身のメールアドレス/パスワードが必要になります。

```plantuml
Actor A
Boundary "J-Quants"

A -> "J-Quants": メールアドレス/パスワード
"J-Quants" -> A: リフレッシュトークン
A -> "J-Quants": リフレッシュトークン
"J-Quants" -> A: IDトークン
A -> "J-Quants": API呼び出し+IDトークン
"J-Quants" -> A: 結果
```

今回、Googleスプレッドシートの関数を使ってこのAPIを呼び出す方法を検討します。スプレッドシートでは、複数のセルに関数が記述されていると、並列計算が行われるため、APIの呼び出しも同時に行われます。リフレッシュトークンやIDトークンは有効期限内で共有できるので、複数のAPI呼び出し間で共有することが望ましいです。そのため、これらのトークンはPropertiesに保存しておくことが効果的です。

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

Properties <- A: メールアドレス/パスワード取得
A -> "J-Quants": メールアドレス/パスワード
"J-Quants" -> A: リフレッシュトークン
A -> Properties: リフレッシュトークン設定
A -> "J-Quants": リフレッシュトークン
"J-Quants" -> A: IDトークン
A -> Properties: IDトークン設定
A -> Properties: API呼び出し+IDトークン

Properties <- B: IDトークン取得
B -> "J-Quants": API呼び出し+IDトークン
```

上記はうまくいっているパターンです。ただし実際はスプレッドシートの計算により、AとBがほぼ同時にリクエストすることが多く、ロックが機能せずに無駄なリクエストが発生してしまいます。

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

Properties <- A: メールアドレス/パスワード取得
Properties <- B: メールアドレス/パスワード取得
A -> "J-Quants": メールアドレス/パスワード
B -> "J-Quants": メールアドレス/パスワード
"J-Quants" -> A: リフレッシュトークン
"J-Quants" -> B: リフレッシュトークン
A -> Properties: リフレッシュトークン設定
B -[#red]> Properties: リフレッシュトークン設定
A -> "J-Quants": リフレッシュトークン
B -> "J-Quants": リフレッシュトークン
"J-Quants" -> A: IDトークン
"J-Quants" -> B: IDトークン
A -> Properties: IDトークン設定
B -[#red]> Properties: IDトークン設定
A -> "J-Quants": API呼び出し+IDトークン
B -> "J-Quants": API呼び出し+IDトークン
```

そこで、PropertiesとA、Bに対してmutexを適用し、この状況を解決していきたいと思います。

# Propertiesを使ったmutex実装

今回はPropertiesにLockをキーとしたPropertyを用意します。このキーがある場合は誰かがリフレッシュトークン/IDトークンを更新している最中なので、更新し終えるまで待機する、と言う仕組みです。このLockキーはちょうどトイレのドアの鍵の役割を果たします。

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

Properties <- A: メールアドレス/パスワード取得
Properties <- B: メールアドレス/パスワード取得
A -> Properties: Lock
activate Properties
Properties <- B: Lockが取得されているので待機
A -> "J-Quants": メールアドレス/パスワード
"J-Quants" -> A: リフレッシュトークン
A -> Properties: リフレッシュトークン設定
A -> "J-Quants": リフレッシュトークン
"J-Quants" -> A: IDトークン
A -> Properties: IDトークン設定
A -> Properties: Unlock
deactivate Properties
Properties <- B: IDトークン取得
A -> "J-Quants": API呼び出し+IDトークン
B -> "J-Quants": API呼び出し+IDトークン
```

素朴に実装するとこんな感じでしょうか。

```javascript:code.gs
function myFunction() {
  const properties = PropertiesService.getScriptProperties();
  const lock = properties.getProperty("lock");
  if (lock) {
    while (properties.getProperty("lock")) {
      Utilities.sleep(1000);
    }
    const idToken = properties.getProperty("idToken");
    //...API呼び出し
    return;
  } else {
    properties.setProperty("lock", Utilities.getUuid());
  }
  // ...リフレッシュトークン取得
  properties.setProperty("refreshToken", refreshToken);
  // ...IDトークン取得
  properties.setProperty("idToken", idToken);
  properties.deleteProperty("lock");
  // ...API呼び出し
}
```

しかしこの実装の場合、AとBが同時にlockを取得した場合にはやはり問題があります。

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

note across: 省略
A -> Properties: properties.getProperty("lock");
B -> Properties: properties.getProperty("lock");
A -> Properties: properties.setProperty("lock", lock);
activate Properties
deactivate Properties
B -[#red]> Properties: properties.setProperty("lock", lock);
activate Properties

note across: 省略
```

Aがロックを設定する前にBがロックを確認してしまったので、Bはまだロックが取得されていないと思ってロックを取得してしまいます。そこで、設定した後にさらに値を取得してそれが自分が設定した値だったら、自分が真のロック取得者とします。

```javascript:code.gs
function myFunction() {
  const properties = PropertiesService.getScriptProperties();
  const lock = properties.getProperty("lock");
  if (lock) {
    while (properties.getProperty("lock")) {
      Utilities.sleep(1000);
    }
    const idToken = properties.getProperty("idToken");
    //...API呼び出し
    return;
  } else {
    const lock = Utilities.getUuid();
    properties.setProperty("lock", lock);
    const currentLock = properties.getProperty("lock");
    if (lock != currentLock) {
      while (properties.getProperty("lock")) {
        Utilities.sleep(1000);
      }
      const idToken = properties.getProperty("idToken");
      // ...API呼び出し
    }
  }
  // ...リフレッシュトークン取得
  properties.setProperty("refreshToken", refreshToken);
  // ...IDトークン取得
  properties.setProperty("idToken", idToken);
  properties.deleteProperty("lock");
  // ...API呼び出し
}
```

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

note across: 省略
A -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
deactivate Properties
B -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
A -> Properties: const currentLock = properties.getProperty("lock");
B -> Properties: const currentLock = properties.getProperty("lock");
A -> A: lock != currentLock 
B <- B: lock == currentLock
A -> A: while (properties.getProperty("lock"))\n  Utilities.sleep(1000);
B -> Properties: リフレッシュトークン/IDトークン設定
B -> Properties: properties.deleteProperty("lock");
deactivate Properties
A -> Properties: IDトークン取得

note across: 省略
```

ちなみにこの実装はコンペア・アンド・スワップ(CAS)として知られています。[^2]

https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%B3%E3%83%9A%E3%82%A2%E3%83%BB%E3%82%A2%E3%83%B3%E3%83%89%E3%83%BB%E3%82%B9%E3%83%AF%E3%83%83%E3%83%97

ところで、次のような状況を考えてみましょう。この場合はAとBが双方ともロックを取ったと勘違いして処理をしてしまいます。特に、Bのロックの最中にAが更新をかけてしまうため、かなり危険です。

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

note across: 省略
A -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
A -> Properties: const currentLock = properties.getProperty("lock");
deactivate Properties
B -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
B -> Properties: const currentLock = properties.getProperty("lock");

A -> A: lock == currentLock 
B <- B: lock == currentLock
B -> "J-Quants": リフレッシュトークン/IDトークン取得
A -> "J-Quants": リフレッシュトークン/IDトークン取得
B -> Properties: リフレッシュトークン/IDトークン設定
A -[#red]> Properties: リフレッシュトークン/IDトークン設定
B -> Properties: properties.deleteProperty("lock");
deactivate Properties
A -> Properties: properties.deleteProperty("lock");

note across: 省略
```

これを解決するために、リフレッシュトークンを設定するタイミングで、現在のロックが自分のものなのかを確認します。

```javascript:code.gs
function myFunction() {
  const lock = Utilities.getUuid();
  const properties = PropertiesService.getScriptProperties();
  const currentLock = properties.getProperty("lock");
  if (currentLock) {
    while (properties.getProperty("lock")) {
      Utilities.sleep(1000);
    }
    const idToken = properties.getProperty("idToken");
    //...API呼び出し
    return;
  } else {
    properties.setProperty("lock", lock);
    const currentLock = properties.getProperty("lock");
    if (lock != currentLock) {
      while (properties.getProperty("lock")) {
        Utilities.sleep(1000);
      }
      const idToken = properties.getProperty("idToken");
      // ...API呼び出し
    }
  }
  // ...リフレッシュトークン取得
  // ...IDトークン取得
  const currentLock1 = properties.getProperty("lock");
  if (lock != currentLock1) {
    while (properties.getProperty("lock")) {
      Utilities.sleep(1000);
    }
    const idToken = properties.getProperty("idToken");
    //...API呼び出し
  } else {
    properties.setProperty("refreshToken", refreshToken);
    properties.setProperty("idToken", idToken);
    properties.deleteProperty("lock");
    // ...API呼び出し
  }
}
```

```plantuml
Actor A
Boundary Properties
Boundary "J-Quants"
Actor B

note across: 省略
A -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
A -> Properties: const currentLock = properties.getProperty("lock");
deactivate Properties
B -> Properties: const lock = Utilities.getUuid();\nproperties.setProperty("lock", lock);
activate Properties
B -> Properties: const currentLock = properties.getProperty("lock");
A -> A: lock == currentLock 
B <- B: lock == currentLock
A -> "J-Quants": リフレッシュトークン/IDトークン取得
B -> "J-Quants": リフレッシュトークン/IDトークン取得
A -> Properties: const currentLock1 = properties.getProperty("lock");
B -> Properties: const currentLock1 = properties.getProperty("lock");
A -> A: lock != currentLock1
B <- B: lock == currentLock1
A -> A: while (properties.getProperty("lock"))\n  Utilities.sleep(1000);
B -> Properties: リフレッシュトークン/IDトークン設定
B -> Properties: properties.deleteProperty("lock");
deactivate Properties
Properties --> A
A -> Properties: IDトークン取得
B -> "J-Quants": API呼び出し+IDトークン
A -> "J-Quants": API呼び出し+IDトークン

note across: 省略
```

さて、コードはごちゃごちゃしてきているので、いくつか重複した部分はまとめると良さそうですね。

# まとめ

GASで複数スレッドがアクセストークンを共有する実装として、Propertiesを用いたmutex実装を紹介しました。久しぶりに並列処理を設計したのでちょっとまとめておきたかったのと、GASで並列処理詳しい人に素人質問してもらいたくて書いてみました。参考になれば幸いです。実装が微妙で気になった方はコメントでツッコミいただけると助かります。

[^1]: トイレの個室を1人で使うことを想定しています。
[^2]: CASは正確には「自身の値を確実に設定する方法」なので、ここではちょっと用途が違うと思う。。。
