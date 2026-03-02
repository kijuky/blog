https://qiita.com/kijuky/items/a42fe5156be58eb15c24

---

GraalVMを使えば、Scala等のJVM言語製アプリケーションをネイティブアプリケーションにできるのは周知の事実です。が、他にもC/C++向けネイティブライブラリを作成できたりもします。便利ですね。

https://www.graalvm.org/latest/reference-manual/native-image/guides/build-native-shared-library/

この記事では、sbtを使ってC/C++向けネイティブライブラリを作る方法を紹介します。

# ゴール

とりあえずこの記事では `Hello World` を標準出力するネイティブライブラリを作ってみます。

# プロジェクト構成

プロジェクトにはsbtを使いました。sbtはscala専用というわけではなく、javaプロジェクトも管理できます。

```properties:project/build.properties
sbt.version = 1.10.10
```

```scala:project/plugins.sbt
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.3.1")
```

```scala:build.sbt
ThisBuild / version := "0.1.0-SNAPSHOT"

ThisBuild / scalaVersion := "3.3.5"

lazy val root =
  project
    .in(file("."))
    .settings(
      organization := "com.example",
      name := "libtestnative",
      libraryDependencies ++= Seq(
        // graalvm
        "org.graalvm.sdk" % "graal-sdk" % "24.1.2" % Provided,
        )
    )
    // assembly
    .settings(
      assembly / mainClass := Some("Main"),
      assemblyPackageScala / assembleArtifact := false,
      // https://stackoverflow.com/q/25144484
      assembly / assemblyMergeStrategy := {
        case PathList("META-INF", "native-image", _*) => MergeStrategy.first
        case PathList("META-INF", _*)                 => MergeStrategy.discard
        case PathList("module-info.class")            => MergeStrategy.discard
        case x =>
          val oldStrategy = (assembly / assemblyMergeStrategy).value
          oldStrategy(x)
      }
    )
```

- 今回はJavaプロジェクトのため、Scalaのバージョンは指定してもしなくてもよいです。ただ、今回は追加していないですが、単体テストをScalaで書く場合はScalaバージョンの指定が必要になります。
    - fat jarにはscalaライブラリは不要なので`assemblyPackageScala / assembleArtiface`に`false`を設定しておきます。
- ネイティブライブラリにするプロジェクトは`lib`を接頭辞とした名前にしておくと、C/C++から参照しやすくなります。
- ネイティブバイナリを作る場合は、fat jarを作るためにsbt assemblyを使います。`assemblyMergeStrategy`としては
    - `META-INF/native-image`配下はGraalVMで使うディレクトリなので保持。
    - それ以外の`META-INF`ディレクトリは無視。
    - `module-info.class`も、fat jarする上では不要なので無視。

# ソースコード

```java:src/main/java/Main.java
import org.graalvm.nativeimage.IsolateThread;
import org.graalvm.nativeimage.c.function.CEntryPoint;

public class Main {
  public static void main(String[] args) {
    // do nothing.
  }

  @CEntryPoint(name = "helloworld")
  public static void helloworld(IsolateThread thread) {
    System.out.println("Hello world");
  }
}
```

- ライブラリであっても `static void` な `main` メソッドが必要です。
- その他のエントリーポイント（外部から呼び出し可能なメソッド）は`CEntryPoint`アノテーションで設定します。
    - アノテーションに設定した名前が、C/C++から呼び出し可能な名前になります。
    - Java側の可視性やメソッド名はJava内でのみ解決されます。
    - メソッドの戻り型や引数の型はいわゆる基本型のみ利用可能です。
        - 文字列をやりとりする場合はchar型の配列でやりとりします。
    - メソッドの引数には必ず`IsolateThread`が必要です。

```properties:src/main/resources/META-INF/native-image/com.example/libtestnative/native-image.properties
Args = --install-exit-handlers \
       --no-fallback \
       --shared
```

- resoucesフォルダ配下に`META-INF/<<organization>>/<<package名>>/native-image.properties`で設定します。
    - ここにGraalVMに設定するコマンドをあらかじめ書いておくと、ビルドスクリプトで省略できます。
    - 同じ階層に`reflect-config.json`や`resource-config.json`を置くと自動で読み取ってくれます。

https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildConfiguration/#embed-a-configuration-file

# ビルドスクリプト

arm系mac osで起動することを前提としています。また、dockerでビルドすることも想定しています。

```build.sh
#!/usr/bin/env sh

set -ex

TARGETOS=${1:-darwin}
TARGETARCH=${2:-arm64}

native-image \
  -jar target/scala-3.3.5/libtestnative-assembly-0.1.0-SNAPSHOT.jar \
  -o libtestnative \
  --link-at-build-time \
  --verbose

mkdir -p "bin/${TARGETOS}-${TARGETARCH}"
if [ "${TARGETOS}" = "darwin" ]; then
  cp libtestnative.dylib "bin/${TARGETOS}-${TARGETARCH}/"
else # linux
  cp libtestnative.so "bin/${TARGETOS}-${TARGETARCH}/"
fi
```

linux向けのビルド用dockerfileです。

```dockerfile:Dockerfile
FROM sbtscala/scala-sbt:eclipse-temurin-23.0.2_7_1.10.10_3.3.5 AS build_jar
WORKDIR /build
COPY . .
RUN sbt assembly

FROM ghcr.io/graalvm/graalvm-community:23.0.2-ol8 AS build_lib
ARG TARGETOS
ARG TARGETARCH
WORKDIR /build
COPY . .
COPY --from=build_jar /build/target/scala-3.3.5/libtestnative-assembly-0.1.0-SNAPSHOT.jar /build/target/scala-3.3.5/
RUN ./build.sh ${TARGETOS} ${TARGETARCH}

FROM scratch
COPY --from=build_lib /build/bin /
```

- jarを作るステージ`build_jar`と、ネイティブライブラリを作るステージ`build_lib`を分けています。
    - これにより、それらのステージで別々のベースイメージを利用できます。`scala-sbt`の全部入りバージョン組み合わせによらずビルドできます。
    - 例えば、`scala-sbt`全部入りバージョン組み合わせだとgraalvmのベースイメージがGLIBCのバージョンが高くてbullseyeに入れられなかったりしますが、ここでgraalvm公式のol8ベースイメージを使うことで、古いGLIBCにリンクしたネイティブバイナリを作成できたりします。
    - 今回の記事はこれを紹介したかったまである。

# ビルド

## arm版mac os

```shell
sbt assembly && ./build.sh
```

カレントディレクトリにヘッダファイルと、`bin/darwin-arm64/libtestnative.dylib`が作成されます。

## arm版linux

```shell
docker build --platform=linux/arm64 --output=bin .
```

`bin/linux-arm64/libtestnative.so`が作成されます。

## amd版linux

```shell
docker build --platform=linux/amd64 --output=bin .
```

`bin/linux-amd64/libtestnative.so`が作成されます。

# 使ってみる

goから使ってみます。

```sample/go.mod
module libtestnative

go 1.23.6
```

```sample/main.go
package main

/*
#cgo CFLAGS: -I${SRCDIR}/..
#cgo LDFLAGS: -L${SRCDIR}/../bin/darwin-arm64 -ltestnative
#include <stdlib.h>
#include <string.h>
#include "libtestnative.h"
*/
import "C"
import (
  "fmt"
  "os"
)

func main() {
    var isolate *C.graal_isolate_t
    var thread *C.graal_isolatethread_t
    if C.graal_create_isolate(nil, &isolate, &thread) != 0 {
        fmt.Fprintf(os.Stderr, "initialization error\n")
        os.Exit(1)
    }
    defer C.graal_tear_down_isolate(thread)

    C.helloworld(thread)
}
```

- `cgo`という機能で、C/C++ライブラリをgoから利用することができます。
    - `LDFLAGS`に渡す`-l`の引数には、`lib`と拡張子を除くファイル名を設定します。そのため、プロジェクト名が`lib`で始まっていると都合が良いのです。
- `graal_isolatethread_t`インスタンスを作成します。ライブラリ実行後に破棄されるように`defer`登録しておきます。

## ビルド

```shell
cd sample
go build
./libtestnative
```

## 出力例

```console
Hello world
```

# まとめ

SBT/GraalVMを使ってC/C++向けネイティブライブラリを作成し、goプロジェクトで利用する例を紹介しました。今回はpure javaでしたが、もちろんscalaロジックを含めることも可能ですので、scalaの複雑なロジックをネイティブライブラリ化してC/C++/Goやその他の非JVM言語から呼び出すことが可能です。メインプロジェクトが非JVM言語で作られていてどうしてもScalaを書きたいScala狂いの人がいたら、プロジェクト内のScala採用の選択肢の一つになれば幸いです。

それではみなさん、良いプログラミングライフ、Scalaライフを！
