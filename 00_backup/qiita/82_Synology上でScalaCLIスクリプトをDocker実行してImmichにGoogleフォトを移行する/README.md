https://qiita.com/kijuky/items/78e2afd53788ebf1a17d

---

# TL;DR

1. SynologyにContainer Managerをインストール
2. ScalaCLIのDockerイメージを使ってスクリプト実行環境を作る
3. Immich APIを叩いてGoogleフォトから写真を移行できるようになった 🎉


# 背景

[昔のPixel端末って、Googleフォトに無制限アップロードできた](https://support.google.com/photos/answer/6220791?hl=ja&co=GENIE.Platform%3DAndroid#zippy=%2Cgoogle-pixel-apixel)んですよね。ところが最新のPixelではその優遇措置がなくなり、気づけば写真でストレージがパンパン…そして追加ストレージの課金が始まると、もう止められない「課金地獄」にハマりがちです。

そこで我が家では、[Synology NAS](https://www.synology.com/ja-jp/products?product_line=ds_j%2Cds_value)上に [Immich](https://immich.app/) を立てて家族みんなで使うことにしました。これでストレージは一気に10数TB、容量不足のストレスから解放！Googleフォト特有のアカウント凍結リスクも減り、快適になりました。

ただし最後の大仕事が残っています。そう、GoogleフォトからImmichへのデータ移行です。
2025年現在、Googleフォトのデータをまとめて取るには [Google Takeout](https://takeout.google.com/settings/takeout/custom/photos?utm_medium=organic-nav&utm_source=google-photos&hl=ja) を使う必要があります。でもそのままドラッグ&ドロップしても、撮影日時などのメタデータが反映されません。実はJSONファイルに分離して保存されているんです。

そこで私は、Google Takeoutの写真＋JSONを突き合わせて、正しいメタデータ付きでImmichにアップロードする[ScalaCLI](https://scala-cli.virtuslab.org/)スクリプトを作りました。

問題は実行環境です。最初はMacからWi-Fi越しにSynologyへアクセスして動かしていたのですが、通信が無駄に重いし安定性もイマイチ。
→ ならば「Synology上で直接スクリプトを走らせよう！」ということで[Docker](https://www.docker.com/ja-jp/)を使いました。

## 1. Container Managerをインストール

まずはSynologyにDocker環境を入れます。

https://www.synology.com/ja-jp/dsm/feature/container-manager

SSHで作業するので、事前にSynologyにSSH接続できるように設定しておきましょう。

https://kb.synology.com/ja-jp/DSM/tutorial/How_to_login_to_DSM_with_root_permission_via_SSH_Telnet

## 2. ScalaCLI docker imageを使う

作業用の共有フォルダを用意して、そこにスクリプトとDocker関連ファイルを置きます。

```yml:compose.yaml
services:
  app:
    build: .
    volumes:
      - .:/app
    working_dir: /app
    command: ["script.sc"]
networks:
  default:
    name: immich_default
    external: true  
```

Immichのdocker-composeで作られたネットワークにアタッチさせるのがポイントです。

```Dockerfile:Dockerfile
FROM virtuslab/scala-cli:1.9.1

RUN apt-get update && apt-get install -y locales-all \
 && rm -rf /var/lib/apt/lists/*

ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

WORKDIR /app
```

ここで日本語ロケールをちゃんと設定しておかないと、フォルダ走査時に日本語ファイル名が全部 `???` になるという悲劇が待っています。忘れがちだけど重要なポイントです。

## 3. ImmichにアップロードするScalaスクリプト例

ScalaCLIから[sttp](https://sttp.softwaremill.com/en/v4.0.12/)を使えば、こんな感じでImmichにアップロードできます。

```scala:script.sc
//> using dep com.softwaremill.sttp.client4::core:4.0.12


import sttp.client4.*
import sttp.client4.httpurlconnection.HttpURLConnectionBackend
import sttp.model.*

...中略...

  val backend = HttpURLConnectionBackend() // http1系でアクセスする
  val response =
    basicRequest
      .header(Header.contentType(MediaType.MultipartFormData)) // これに限らず、immich apiはちゃんとContent-Typeを指定しないと4xxが返る。
      .header("x-api-key", IMMICH_TOKEN) // ユーザー設定からAPIトークンを作成しておく
      .header("x-immich-checksum", checksum) // checksumに写真のSHA1を設定しておくと重複確認してくれる
      .post(uri"http://immich_server.immich_default:2283/api/assets") // ここが重要
      .multipartBody(assetData, deviceAssetId, deviceId, fileCreatedAt, fileModifiedAt, filename, isFavorite)
      .send(backend)
```

ポイントは コンテナネットワーク越しのアクセス方法です。

```
http://<コンテナ名>.<ネットワーク名>:<内向きのポート番号>/<APIのパス>
```

[Immich公式のdocker-compose (dev版)](https://github.com/immich-app/immich/blob/main/docker/docker-compose.yml) なら、上の例のように`http://immich_server.immich_default:2283/api/assets` になります。

## 4. Happy :tada:

これで Synology上でScalaCLIスクリプトを動かして、GoogleフォトからImmichへの移行が完了しました！
もうMacから無線越しでゴニョゴニョする必要なし。NAS内で完結してスッキリです。

# まとめ

- Googleフォト無制限が終わり、課金地獄を避けるためImmichへ移行
- Google TakeoutのJSONを使って撮影日時などのメタデータも移行可能
- ScalaCLIスクリプトをDockerで動かすことで、Synology上にScala環境を入れなくてもOK

これを応用すれば、Immich移行だけでなく「Synology上の自動処理をScalaで書いて、Dockerで実行する」なんて使い方もできます。

それでは、皆さんも良いScalaライフを！

# 参考文献

https://qiita.com/kapyba_ch/items/364f68807ad3a0d6f763

https://blog.3qe.us/entry/2024/02/29/212529
