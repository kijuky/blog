# blog

## 使い方

ディレクトリをブログタイトル、そのディレクトリ直下にREADME.mdを置き、ブログ本文を内容にします。こうすることで、WebのGitHub上から、ディレクトリを指定するとブログ本文が描画されます。各記事がWebUI上でソートされるようにするため、ディレクトリ名はASCIIでソートできるように命名します。

ディレクトリにはそのブログのメタ情報としてmeta.yamlを置けます。属性はすべてオプショナルですが、title, published_at, modified_atなどがあります。git commitのタイミングでmeta.yamlを更新するためのフックとしてpre-commitフックを用意しています。

※pre-commitフックはscala-cli製のため、実行には[scalaのインストール](https://www.scala-lang.org/download/)が必要です。

```shell
git config --local core.hooksPath .githooks
```

## 特殊なフォルダ

### 00_backup

以前のブログサイト毎にディレクトリを作成して、そこにブログ記事の内容を置いています。
