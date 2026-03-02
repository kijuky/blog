https://qiita.com/kijuky/items/2f06e4a7f1bf12140d10

---

# はじめに
iOS ライブラリのテストカバレッジを取るのに苦戦したので、まとめる。

# Jenkins を用いたカバレッジレポートの出力

[Cobertura Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Cobertura+Plugin) を使ってカバレッジを出力します。
Objective-C のカバレッジツールで Cobertura XML 形式で結果を出力し、
結果をプラグインに読み込ませることでグラフ化できます。

## 事前準備

プロジェクトの設定で、下記のオプションを YES にします。

- Generate Test Coverage Files (Xcode 7 以降は Generate Leagcy Test Coverage Files)
- Instrument Program Flow

ただし、Xcode 7 でこの設定を使うと、まれにテスト実行後に落ちることがあります（[ここ](https://forums.developer.apple.com/thread/9765)とか、[ここ](https://github.com/jonreid/XcodeCoverage/issues/40)とかを参照）。
そのため、テスト「実行前」に下記を叩いておくのが無難です。
※slather については後述

```shell-session
slather setup path/to/xcodeproj
```

## [XcodeCoverage](https://github.com/jonreid/XcodeCoverage) を使う場合

本家をみると、標準的な方法と CocoaPods を利用する方法の２種類があります。
ここでは、標準的な方法を例に環境を作る方法を紹介します。

### Xcode 6 以前

1. 適当な場所で XcodeCoverage をクローンする。

    ```shell-session
    git clone https://github.com/jonreid/XcodeCoverage.git
    chmod -R +x XcodeCoverage/lcov-*/bin
    ```

2. Xcode のプロジェクト設定の Run Script に XcodeCoverage/exportenv.sh を実行するように設定する。

3. xcodebuild でテストを実行する。

    ```shell-session
    xcodebuild clean test -project path/to/xcodeproj -scheme scheme -destination 'platform=iOS Simulator,name=iPhone6s'
    ```

4. XcodeCoverage/getcov でカバレッジを取得する。

    ```shell-session
    XcodeCoverage/getcov --xml
    ```

5. [gcovr](http://gcovr.com/) でカバレッジ結果を Cobertura XML 形式に変換する。

    ```shell-session
    brew install gcovr
    gcovr 
    ```

### Xcode 7 以降

1. について
   lcov のプログラムが足りないため、[lcov](http://ltp.sourceforge.net/coverage/lcov.php) から足りないプログラムを持ってくる必要があります（[ここ](https://github.com/jonreid/XcodeCoverage/issues/40)を参照）。

    ```
    git clone https://github.com/jonreid/XcodeCoverage.git
    git clone https://github.com/linux-test-project/lcov.git
    cp lcov/bin/* XcodeCoverage/lcov-*/bin
    ```

2. について
   カバレッジデータが出力される場所が変わったため、パスを変更する必要があります。

3. 以降は Xcode 6 と同様。

## [Slather](https://github.com/venmo/slather) を使う場合

1.  xcodebuild でテストを実行する。
2.  slather でカバレッジ結果を出力する

    ```shell-session
    slather --cobertura-xml --build-directory path/to/gcda path/to/xcodeproj
    ```

# 開発中のカバレッジ取得

## Xcode 6 以前

### XcodeCoverage を使う場合

getcov に --show オプションを追加するとカバレッジレポートを取得できます。

### Slather を使う場合

slather に --html --show オプションを追加するとカバレッジレポートを取得できます。

## Xcode 7 以降

1. Enable Coverage を YES にします。

2. Enable Coverage にチェックを入れます（各環境ごとに必要です！）

3. テストを実行すると、コード上にカバレッジ結果が表示されます。

4. レポートとしてみたい場合


# 最後に

カバレッジが取得できるようになると、テストコードを書くモチベーションが上がるので、ぜひ環境構築したいところですね。

若干ややこしいですが、トライしてみる価値はあると思います。
