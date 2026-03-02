https://qiita.com/kijuky/items/046e23e647e738d41d4c

---

[GraalVM を使って Scala コードのネイティブイメージを作る](https://qiita.com/__kijuky__/items/03bb1f65545f9418a13e) の続きです。

こちらの記事でネイティブイメージを作成したとき、下記の警告が出ていました。

```shell
Warning: Reflection method java.lang.Class.getMethods invoked at scala.Enumeration.populateNameMap(Enumeration.scala:201)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.runtime.Statics$VM.findUnsafe(Statics.java:178)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.getFields$1(Enumeration.scala:195)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.populateNameMap(Enumeration.scala:197)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.getFields$1(Enumeration.scala:195)
Warning: Aborting stand-alone image build due to reflection use without configuration.
```

こちらの警告が出ないようにするための対処方法を共有します。

# この警告は何？

GraalVM の `native-image` はリフレクションのコードに完全に対応しておらず、非対応のリフレクションのコードが出てきた場合にこの警告が表示されます。この警告を出さないようにするために、`native-image` に対してどんなリフレクションの機能を使っているかを教えてあげる必要があります。

# プロファイルの作成

その情報を作成する方法が下記です。

```
% mkdir -p META-INF/native-image
% java -agentlib:native-image-agent=config-output-dir=META-INF/native-image -cp .:/usr/local/opt/scala/libexec/lib/scala-library.jar Test
% ls META-INF/native-image 
jni-config.json			resource-config.json
proxy-config.json		serialization-config.json
reflect-config.json
```

参考までに各ファイルはこのような内容でした

```json:jni-config.json
[
]

```

```json:proxy-config.json
[
]

```

```json:reflect-config.json
[
{
  "name":"java.lang.invoke.VarHandle",
  "methods":[{"name":"releaseFence","parameterTypes":[] }]
}
]

```

```json:resource-config.json
{
  "resources":{
  "includes":[{"pattern":"\\Qlibrary.properties\\E"}]},
  "bundles":[]
}

```

```json:serialization-config.json
[
]

```

これらのファイルが存在した状態で再度コンパイルしてみます。

# コンパイル

```shell
% native-image Test -cp .:/usr/local/opt/scala/libexec/lib/scala-library.jar
[test:6158]    classlist:   1,292.99 ms,  0.96 GB
[test:6158]        (cap):   2,673.58 ms,  0.96 GB
[test:6158]        setup:   3,875.80 ms,  0.96 GB
[test:6158]     (clinit):     171.38 ms,  1.69 GB
[test:6158]   (typeflow):   5,738.16 ms,  1.69 GB
[test:6158]    (objects):   5,708.21 ms,  1.69 GB
[test:6158]   (features):     356.71 ms,  1.69 GB
[test:6158]     analysis:  12,223.73 ms,  1.69 GB
[test:6158]     universe:     331.86 ms,  1.69 GB
[test:6158]      (parse):     724.78 ms,  1.69 GB
[test:6158]     (inline):   1,821.46 ms,  1.71 GB
[test:6158]    (compile):   3,647.49 ms,  3.27 GB
[test:6158]      compile:   6,774.16 ms,  3.27 GB
[test:6158]        image:     934.15 ms,  3.27 GB
[test:6158]        write:     301.58 ms,  3.27 GB
[test:6158]      [total]:  25,898.56 ms,  3.27 GB
```

今度は警告なしでコンパイルできました。めでたしめでたし。

# 参考

- [Introducing the Tracing Agent: Simplifying GraalVM Native Image Configuration](https://medium.com/graalvm/introducing-the-tracing-agent-simplifying-graalvm-native-image-configuration-c3b56c486271)
