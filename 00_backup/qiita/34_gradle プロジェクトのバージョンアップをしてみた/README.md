https://qiita.com/kijuky/items/6efe0706403d41893243

---

みなさん、ごきげんよう。最近（と言ってもちょい前だけど） kotlin の 1.6 が出たので、せっかくなら業務で触ってみたいと思ったので、手元の gradle + kotlin プロジェクトのバージョンアップをしてみました。バージョンアップでちょっとつまづいたところがあったので、メモっておきます。

# 結論

対象プロジェクトの主要なライブラリのバージョン変更前後をまとめました。

| ライブラリ | 変更前 | 変更後 | 備考 |
|:-:|:-:|:-:|:-:|
| kotlin | 1.5.10 | 1.6.10 | |
| gradle | 7.1 | 7.3.2 | |
| spring-boot | 2.5.1 | 2.5.7 | |
| springfox | 2.8.0 | 3.0.0 | |
| postgresql | 9.5 / 9.6.9 | 11.9 | 開発環境で別のバージョンが指定されてた |

## springfox のバージョンアップについて

[springfox](https://github.com/springfox/springfox) とは、アノテーションベースの swagger/openapi 定義プラグインです。バージョン 3.0.0 になって依存の書き方が変わったので、そこだけ注意です。

```gradle
//implementation('io.springfox:springfox-swagger2:2.8.0')
//implementation('io.springfox:springfox-swagger-ui:2.8.0')
//implementation('io.springfox:springfox-core:2.8.0')
implementation('io.springfox:springfox-boot-starter:3.0.0')
```

いわゆる bom が提供された感じですかね。1行で済むので楽になりました。

## spring-boot のバージョンアップについて

現時点(2021年12月)で、spring-boot の最新バージョンは 2.6.1 なのですが、[springfox のバージョンが 3.0.0 の場合、spring-boot のバージョンを 2.6.x にあげられない不具合があります](https://github.com/springfox/springfox/issues/3791)。仕方がないので 2.5 系の最終バージョンを設定しました。

https://github.com/springfox/springfox/issues/3791

## junit のバージョンアップについて

ちょっと詳しく調べてませんが、 junit4 の依存がなくなったため、テストは junit5 で書く必要があります。

## postgresql のバージョンアップについて

[最新の flyway のバージョンでは、postgresql のバージョン 9 系はサポートされなくなりました（有償でのサポートはある）](https://flywaydb.org/documentation/database/postgresql)。そのため（無料で頑張るなら） postgresql のバージョンは最低でも 10 系にする必要がありそうです。

https://flywaydb.org/documentation/database/postgresql

また、10 系からは `POSTGRES_HOST_AUTH_METHOD=trust` がないと GitLab CI とかで失敗するようです。

https://qiita.com/tomopict/items/23cc4332781bb28d2973

# 感想

最近はプロジェクトのフレームワーク/ミドルウェアのバージョンを上げることが多いのですが、たくさんのライブラリに依存しているといい感じの組み合わせを見出すのが大変ですね... 加えて、バージョンを揃えたところでやっとコンパイルが走るので、バージョンを揃えることってコンパイルの前段階の作業であり、バージョンアップによるIFの変更対応はこれから、と思うと骨が折れますね。皆さんも直近でライブラリの更新トリガーが入ったと思うので、一緒に頑張りましょう。

現場からは以上です。

そうだった、このバージョンアップが終わったら kotlin の最新機能とか使っていきたいと思います。
