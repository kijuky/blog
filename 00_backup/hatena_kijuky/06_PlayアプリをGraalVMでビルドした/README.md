https://kijuky.hatenablog.com/entry/2022/04/28/105127

---

みたいな記事を書いた。

[https://qiita.com/kijuky/items/ab3d1aecc9e4466145f9:embed:cite]

もともと[パーサーコンビネーターがサクッとネイティブ化できた](https://qiita.com/kijuky/items/21be8d13a89a4b835ad4#%E3%83%8D%E3%82%A4%E3%83%86%E3%82%A3%E3%83%96%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%82%92%E4%BD%9C%E3%82%8B)ので、最近だとネイティブサポートアツいのかな？と思って、軽い気持ちでPlayに挑戦したら丸２日かかった。ただ、[公式ドキュメントが日本語で参照できる](https://docs.oracle.com/cd/F44923_01/enterprise/21/docs/reference-manual/native-image/)ので、日本人にとってはとてもやりやすい環境だなと思った。

周辺を調べていると、[SpringBoot](https://spring.io/projects/spring-boot)は[SpringNative](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/)というプラグインでネイティブ化をサポートしているみたい。面白いなと思ったのはそのアプローチで、SpringのAOTの解析結果を基に reflect-config.json を作っちゃうという（[参照](https://www.infoq.com/jp/news/2021/05/spring-native-beta-available/)）。一番めんどくさい reflect-config.json をこんな方法で解決しちゃうとは、賢いですね。

一方で Scala 界隈では [akka-graal-config](https://github.com/vmencik/akka-graal-config) という形で、事前に作った reflect-config.json をまとめたリポジトリを作って解決しようとしています。これは公式でも紹介している方法ですね。

いずれにせよ、reflect-config.json が解決すれば、Play ってネイティブ化できるんですよね。その熱が冷めないうちに本家に discussion 立てちゃいました。

[https://github.com/playframework/playframework/discussions/11253:embed:cite]

本格的に対応するとなると、もっとちゃんと reflect-config.json を整理する必要があるとは思うけど...
