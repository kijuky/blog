https://qiita.com/kijuky/items/03bb1f65545f9418a13e

---

今日やったことのメモです。

# 1. GraalVM のインストール

## sudo できる場合

homebrew でインストールできます。

```shell
brew install --cask graalvm/tap/graalvm-ce-java11
```

GraalVM の I/F は JDK と同じです。そのためコンパイルには `javac` を利用します。また、パスの通し方も JDK と一緒で、 `java_home` コマンドを使って変更できます。今回は java11 ベースのものをインストールしたので、 java11 のパスを通すようにすると GraalVM が使われるようになります。

```shell
/usr/libexec/java_home -v 11
```

参考：https://github.com/graalvm/homebrew-tap

## sudo できない場合

あなたが権限を持っている適当なディレクトリに GraalVM を解凍します。

```shell
curl -L https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-21.0.0/graalvm-ce-java11-darwin-amd64-21.0.0.tar.gz | tar zxv
```

パスを通しておきます。

```shell:.envrc
export PATH="~/graalvm-ce-java11-21.0.0/Contents/Home/bin:$PATH"
export JAVA_HOME="~/graalvm-ce-java11-21.0.0/Contents/Home"
```

参考：https://www.graalvm.org/docs/getting-started/macos/

## 署名を修正する

ダウンロードしてきたファイルは（おそらく）提供しているビルドマシンの署名がついているため、そのまま実行できません（「壊れたファイルです」というダイアログが表示される）。そこで、その署名を削除します。

```shell
xattr -r -d com.apple.quarantine /Library/Java/JavaVirtualMachines/graalvm-ce-java11-21.0.0         
```

指定するのは `graalvm-ce-java11-21.0.0` フォルダです。それ以上でも、それ以下でもダメです。

## 確認

試しに `javac --version` してみて、バージョン情報が出たらインストール終了です。

```shell
% javac --version
javac 11.0.10
```

# 2. Scala のインストール

homebrew でインストールできます

```shell
brew install scala
```

# 3. "Hello World" をコンパイル

適当な Hello World を用意します。

```scala:Test.scala
object Test extends App {
  println("Hello World")
}
```

コンパイルして、class ファイルを生成しておきます。

```shell
scalac Test.scala
```

GraalVM Updater `gu` から、 `native-image` をインストールします。 `gu` は先程インストールした GraalVM の PATH 上にいるはずです。

```shell
% gu install native-image
Downloading: Component catalog from www.graalvm.org
Processing Component: Native Image
Downloading: Component native-image: Native Image  from github.com
Installing new component: Native Image (org.graalvm.native-image, version 21.0.0)
```

`native-image` は class ファイルからネイティブイメージ（実行可能ファイル）を作成するためのコンパイラです。scala のネイティブイメージを作成する場合は scala-library.jar をクラスパス上に入れておきます。

```shell
% native-image Test -cp .:/usr/local/opt/scala/libexec/lib/scala-library.jar
[test:2206]    classlist:   1,080.45 ms,  0.96 GB
[test:2206]        (cap):   2,938.76 ms,  0.96 GB
[test:2206]        setup:   4,147.52 ms,  0.96 GB
[test:2206]     (clinit):     176.38 ms,  1.69 GB
[test:2206]   (typeflow):   5,747.18 ms,  1.69 GB
[test:2206]    (objects):   5,699.12 ms,  1.69 GB
[test:2206]   (features):     246.20 ms,  1.69 GB
[test:2206]     analysis:  12,078.51 ms,  1.69 GB
[test:2206]     universe:     317.66 ms,  1.69 GB
Warning: Reflection method java.lang.Class.getMethods invoked at scala.Enumeration.populateNameMap(Enumeration.scala:201)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.runtime.Statics$VM.findUnsafe(Statics.java:178)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.getFields$1(Enumeration.scala:195)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.populateNameMap(Enumeration.scala:197)
Warning: Reflection method java.lang.Class.getDeclaredFields invoked at scala.Enumeration.getFields$1(Enumeration.scala:195)
Warning: Aborting stand-alone image build due to reflection use without configuration.
Warning: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
[test:2232]    classlist:     720.17 ms,  0.96 GB
[test:2232]        (cap):   1,760.67 ms,  0.96 GB
[test:2232]        setup:   2,684.29 ms,  0.96 GB
[test:2232]     (clinit):     154.05 ms,  1.22 GB
[test:2232]   (typeflow):   5,254.60 ms,  1.22 GB
[test:2232]    (objects):   5,335.18 ms,  1.22 GB
[test:2232]   (features):     177.53 ms,  1.22 GB
[test:2232]     analysis:  11,089.90 ms,  1.22 GB
[test:2232]     universe:     276.48 ms,  1.22 GB
[test:2232]      (parse):     750.87 ms,  1.69 GB
[test:2232]     (inline):     880.71 ms,  1.69 GB
[test:2232]    (compile):   3,383.42 ms,  2.33 GB
[test:2232]      compile:   5,547.63 ms,  2.33 GB
[test:2232]        image:     809.69 ms,  2.33 GB
[test:2232]        write:     245.06 ms,  2.33 GB
[test:2232]      [total]:  21,492.75 ms,  2.33 GB
Warning: Image 'test' is a fallback image that requires a JDK for execution (use --no-fallback to suppress fallback image generation and to print more detailed information why a fallback image was necessary).
```

# 4. 確認

```shell
% ./test
Hello World
```

# 所感

- native-image 作成にマシンによっては10分くらいかかるので、普段は scalac だけすると良さそう。
- 生成されるバイナリが約 9MB だった。 Xcode のデフォルトで生成されるC言語の Hello World(デバッグビルド版)が約 69KB なので、およそ 100 倍以上の容量になった。でも、ネイティブで Scala が書けるならペイするよね？

# 参照

- https://qiita.com/erin/items/3a6c6a3fd753eef11ae2

# 追記

- この記事は、当時使っていた「sudo が使えない会社貸与 Mac」を前提に試した内容です。
- 現在は `sbt` と GraalVM の Docker イメージを組み合わせる方法のほうが、環境差分を減らせて楽です。
- この時点では fallback image の挙動を理解できておらず、結果として「JDK が必要な実行ファイル」が生成されています（回避方法は次の記事）。

続きます -> [GraalVM を使って Scala コードのネイティブイメージを警告なしで作る](https://qiita.com/__kijuky__/items/046e23e647e738d41d4c)
