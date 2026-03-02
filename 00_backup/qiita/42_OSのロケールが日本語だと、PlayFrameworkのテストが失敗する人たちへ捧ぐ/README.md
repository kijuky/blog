https://qiita.com/kijuky/items/2976f429260e4e6136ce

---

現在の main ブランチ (a06ddb0) にて、OSのロケールが日本語だとテストが失敗するようです。

```
% sbt -error test
...(省略)
[error]     x when it is located in a subform (and sub-subform) and returns an error it should automatically prefix the error key with the parent form field (45 ms)
[error]      '0 から 1 の間のサイズにしてください' != 'size must be between 0 and 1' (FormSpec.scala:1507)
[error] Actual:   0 から 1 の間のサイズにしてください
[error] Expected: size must be between 0 and 1
[error]     x when it is located in a subform (and sub-subform) and returns an error it should automatically prefix the error key with the parent form field (14 ms)
[error]      '0 から 1 の間のサイズにしてください' != 'size must be between 0 and 1' (FormSpec.scala:1507)
[error] Actual:   0 から 1 の間のサイズにしてください
[error] Expected: size must be between 0 and 1
...(省略)
[error] Failed: Total 175, Failed 2, Errors 0, Passed 173
[error] Failed tests:
[error] 	play.data.CompileTimeDependencyInjectionFormSpec
[error] 	play.data.RuntimeDependencyInjectionFormSpec
[error] (Play-Java-Forms / Test / test) sbt.TestsFailedException: Tests unsuccessful
```

ベーススペックの `FormSpec.scala` に定義されたテストで失敗しているようです。

余談ですが、スペックの継承って結構難しいなと思いました。以前仕事でそういう仕組みを作ったことがあるんですが、指数関数的にスペック数が増えるので、管理が大変でした。。

# 前提

## OS

macOS

## Shell

zsh

## locale

```
% locale                                           
LANG="ja_JP.UTF-8"
LC_COLLATE="ja_JP.UTF-8"
LC_CTYPE="ja_JP.UTF-8"
LC_MESSAGES="ja_JP.UTF-8"
LC_MONETARY="ja_JP.UTF-8"
LC_NUMERIC="ja_JP.UTF-8"
LC_TIME="ja_JP.UTF-8"
LC_ALL=
```

日本語ロケールです。

# ロケールを変更する

Java ベースのソフトウェアはロケールの変更方法が複数あって、かなり戸惑いました。

## (NG) LC_ALL を変更する

```
% LC_ALL=C sbt test
```

OS のロケール設定をいじる方法です。sbt より手前に `LC_ALL=C` とすることで、英語のロケールになります。が、sbt ではこれだと反映されませんでした。

## (NG) システムプロパティを設定する

```
% sbt -J-Duser.language=en -J-Duser.country=US test
```

システムプロパティを設定します。JVM に設定を渡すには `-J` が必要です。しかしながら、これも反映されませんでした。

## (OK) JAVA_TOOL_OPTIONS を設定する

```
% JAVA_TOOL_OPTIONS=-Duser.language=en sbt test  
```

https://www.gwtcenter.com/acquiring-and-setting-locale

`JAVA_TOOL_OPTIONS` 環境変数に設定することで反映されました。sbt の出力を見ていると、こんな出力もあったので、なんか認識されているようです。

```
Picked up JAVA_TOOL_OPTIONS: -Duser.language=en
```

# まとめ

ロケールの変更で困ったら `JAVA_TOOL_OPTIONS` に設定しましょう。
