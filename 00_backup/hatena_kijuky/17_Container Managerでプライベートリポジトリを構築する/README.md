https://kijuky.hatenablog.com/entry/2024/03/20/225947

---

Synology NASでプライベートクラウドを構築する話。NASでDockerが動くらしいので、とりあえず環境構築しておけばいい感じのクラウド環境になりそう。買ったやつはまだメモリにも余裕があるし、足りなくなったら増設すれば良い。

環境構築は下記のサイトを参考にした。一部分設定が異なるので、差分だけメモっておく。気が向いたらそのうちQiitaに書く。NASあまりにも便利すぎてみんな使いだして品薄になっても困るけど。

https://taktak.jp/2019/06/09/4060/

https://bitkatasumi.com/2024-01-02-docker-registry/

まずはパッケージマネージャーからContainer Managerをインストールする。

次に自己証明書ではなくLet'sEncriptから証明書をダウンロードする。プライベートリポジトリといえど、https通信が必須らしく、そのためには正規の証明書が必要らしい。この証明書を得るためにDDSNの設定が必要になるので、DDSNの設定がまだの場合は、それからしておく。

次にリポジトリ用のdockerコンテナを立てる。Container Managerを作ると、永続用に"docker"という共有フォルダが作られる。ここに下記のようなファイルとフォルダを作成する。

ファイル：/docker/registry/config/config.yml

```dockerfile
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true     # open delete api
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
auth:
  htpasswd:
    realm: basic-realm
    path: /auth/htpasswd    # use apache basic-auth
```

ファイルの書式の詳細はこちら

https://docs.docker.jp/registry/configuration.html#list-of-configuration-options

ファイル：/docker/registry/config/htpasswd

レジストリにログインするユーザーとそのパスワードを保管するためのファイル。このファイルを作るための専用のdockerイメージがあるので、サクッとdocker runして作ることができる。

```shell
docker run --rm -ti xmartlabs/htpasswd ユーザー名 パスワード > htpasswd
```

イメージの詳細はこちら

https://github.com/xmartlabs/docker-htpasswd

フォルダ：/docker/registry/certs

ここに証明書のファイル(.crt, .key)をいれる。Synologyのコントロールパネル > セキュリティ > 証明書から、証明書のエクスポートで各種 pem ファイルが圧縮されたzipファイルが降ってくるので、それを解凍して、次のコマンドで .crt と .key ファイルを作成する。

```shell
cp privkey.pem domain.key
awk '{print $0}' cert.pem chain.pem > domain.crt
```

なお、証明書が更新されたタイミングでこれを行う必要がある。Let'sEncryptは3ヶ月しかもたないので、そのたびにやる必要がある。ダルい。なんとかして自動化しなければ...(Synology NASはWebAPIがあるらしいので、自動化も頑張ればできるかも？）

https://kb.synology.com/en-global/DG/DSM_Login_Web_API_Guide/2

フォルダ：/docker/registry/images

ここにプッシュしたイメージが登録される。

ここまでできたらContainre Managerからコンテナを作成する。各種設定について

- 自動再起動はONにしておく
- デフォルトポートはSynology NASコンソールが使っているので5000, 5001を避けて設定する（例：5002）
- 下記ボリュームの設定をする
- /docker/registry/certs:/certs
- /docker/registry/images:/var/lib/registry
- /docker/registry/config/htpasswd:/auth/htpasswd
- /docker/registry/config/config.yml:/etc/docker/registry/config.yml
- 下記環境変数を追加する
- REGISTRY_HTTP_TLS_KEY
  - /certs/domain.key
- REGISTRY_HTTP_TLS_CERTIFICATE
  - /certs/domain.crt

ここまでできたらコンテナを起動し、下記で動作確認する

```shell
# Macのコンソールで実行
curl https://testuser:testpass@example.synology.me:5002/v2/_catalog

# 下記のようなレスポンスが返る
{"repositories":[]}
```

次に、このレポジトリをNASに登録してプライベートリポジトリに上げたイメージからコンテナを作れるようにする。

レジストリのURLは https://IPアドレス:さっき開けたポート(例:5002) になる。