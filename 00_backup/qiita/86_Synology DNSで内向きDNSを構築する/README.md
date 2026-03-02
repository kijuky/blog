https://qiita.com/kijuky/items/d39ae691725ad4e93f0c

---

:::note info
**これで何ができるか**
家庭内LANにおいて、名前でパソコンにアクセスできます。
:::

:::note warn
DNSはIPアドレスまでしかマッピングしないので、ポート番号も含めてマッピングしたい場合はリバースプロキシの導入を検討してください。なお、[SynologyDSMはリバースプロキシを搭載済み](https://kb.synology.com/ja-jp/DSM/help/DSM/AdminCenter/system_login_portal_advanced?version=7)ですので、家庭内のDNSは一旦全てSynologyで受けてから、リバースプロキシでサービスに飛ばすようにすると良いです。
:::

家庭内LANを構築しているとIP直打ちでサービスにアクセスすることが多いです。しかし、サービスが多くなってくるとIPアドレス直打ちは面倒なので、名前でアクセスしたくなります。そこで今回はDNSを使って名前からIPアドレスを解決できるようにする環境を構築します。

# 前提条件

- NASとして[Synology](https://www.synology.com/ja-jp)を持っていること（私の環境は Synology DS224+です）
- Synologyに[DNS Server](https://kb.synology.com/ja-jp/DSM/help/DNSServer/dns_server_desc?version=7)をインストールしていること

# 1. ゾーン名を決める。

「ゾーン」とは、ホスト名の hoge. **com**とか fuga. **co.jp** の部分のことを指します。特にこだわりがなければ、下記のどれかにします。

- home.arpa ([RFC8375](https://tex2e.github.io/rfc-translater/html/rfc8375.html))
- intranet, internal, private, corp, home, lan ([RFC6762](https://tex2e.github.io/rfc-translater/html/rfc6762.html))

逆に、これらの名前を設定してはいけません

- local (mDNS)
- com などのグローバルなやつと衝突する

# 2. プライマリゾーンを作成

画像の通りにプライマリゾーンを作成していきます。ゾーン＞作成＞プライマリゾーン

![スクリーンショット 2026-02-25 15.05.12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/e7b20ac5-50e4-413b-a5f5-949b94d7819d.png)

ドメイン名に先ほど決めたゾーン名を、プライマリDNSサーバーにはルーター(DHCPサーバー)のIPアドレスを設定します。

![スクリーンショット 2026-02-25 15.07.56.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/307c295d-83ec-4508-b0c0-c91b9bdda3f5.png)

その他の設定は特にいじらなくて良いです。「保存」を押下します。

# 2. リソースレコードを追加

ゾーン＞作成したゾーンを右クリック＞リソースレコードを押下します。

![スクリーンショット 2026-02-25 15.16.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/012ad636-506f-46d6-b984-48d87395b961.png)

作成＞Aタイプを押下します。

※AタイプはIPv4のアドレスを返します。もしIPv6のアドレスを返したい場合はAAAAタイプを押下します。ただし、通常はその必要はないはずです。

![スクリーンショット 2026-02-25 15.16.14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/bffbb4bb-53bd-40b8-965d-f670ef2937b7.png)

名前に、アクセスするサービス名を、IPアドレスにそのサービスが稼働しているPC（もしくはそのサービスに到達可能なリバースプロキシが稼働しているPC）を設定します。

（ちなみに [dashy](https://dashy.to/) はダッシュボードサービスです）

![スクリーンショット 2026-02-25 15.20.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/e45afc4a-fe5a-4394-88e7-84d1063f12cf.png)

# 3. 転送設定を有効にする

内向き(home.arpa)以外の名前はグローバルDNSに解決してもらうことにします。解決＞フォワーダを有効にするにチェックを入れます。

フォワーダ1にルーター(DHCPサーバー)のIPアドレスを設定します。

転送ポリシーは「転送専用」にします。

![スクリーンショット 2026-02-25 15.24.13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/80d8ca3a-73d3-48d9-a7e2-503e1da54039.png)

「適用」を押下します。

# 4. ルーター(DHCPサーバー)でDNSを配信する

ルーター(DHCPサーバー)でDNSを設定します。ここでは[TP-Link](https://www.tp-link.com/jp/)を例に取ります。プライマリDNSとセカンダリDNSの両方にSynology(DNS Server)のアドレスを設定します。

※TP-Linkの場合、省略すると自身のアドレス(ここでは192.168.0.1)がDNSとして設定されます。その結果、プライマリDNSでは解決できてセカンダリDNSでは解決できなくなるなど、挙動が安定しなくなります。

![スクリーンショット 2026-02-25 15.26.48.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/58bbc959-c9d1-401c-8e4c-133289d31f7d.png)

# おしまい

お疲れ様でした。これで安定稼働すれば成功です！
