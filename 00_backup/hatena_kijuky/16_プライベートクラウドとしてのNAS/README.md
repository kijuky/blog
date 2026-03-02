https://kijuky.hatenablog.com/entry/2024/03/18/131124

---

弊社のテックトーク過去回を見ていたら、これを見かけました。

https://youtu.be/Uh72KemCTKE?si=obTOyEkDcN-7ZodO

https://kakaku.com/item/K0001553981/

見終わったら、もう紹介されていたNASをポチっていました。最近のNASはLANに繋がった外付けHDDってだけではなく、ちょっとしたプライベートクラウドとして使えるのですね。便利な世の中です。

### Synology Photos

このNASを買う前から、自宅でNASを運用していました。NASの用途は主に写真・動画のバックアップで、[nasne ACCESS](https://manuals.playstation.net/document/jp/nasne/nasneaccess/index.html)を使ったり、[RAVPower FileHub](https://www.ravpower.jp/shop/gear/filehub/filehub-filehub/rp-wd009/)に外付けHDDを繋げて、そこに[PhotoSync](https://play.google.com/store/apps/details?id=com.touchbyte.photosync&hl=ja&gl=US&pli=1)の[NASアドオン](https://play.google.com/store/apps/details?id=com.touchbyte.photosync.nas&hl=ja&gl=US)を使って写真・動画をバックアップしていました。しかし、nasneは本体HDDがぶっ壊れたし、FileHubは稀に接続が途切れてバックアップできなかったりしていて、ちょっと不満がありました。

SynologyのNASは常時起動を前提としているため、安定感抜群です。そしてSynology Photosというソフトウェアを使うことで写真を自動バックアップできるようになりました。ただバックアップするだけではなく、GoogleフォトのようなUIが提供されているので、もうバックアップはこれメインでいいような気がしています。

Googleフォトは共有機能が強いですが、保存容量が高く(100GB 2,500円/年)、動画の容量制限がある(10GB未満)、容量無制限の場合はPixel 5以下で画質を落とす必要ありなど、いろいろ制約があります。Synology Photosを使うと、保存容量はHDDを買えば好きなだけ増やせますし、スマホもPixel 5以外の選択が増えます。後述するVPNを構築すれば、外出先からも写真のバックアップや共有ができます。

### VPN Server

Synology NASには無料でDDNS機能とVPN機能を追加できます。そのため、外部のネットワークからNASに安全にアクセスできます。VPNの方式は現状OpenVPN一択です。ただ、NASの構築にあわせてルーターも新調し、VPNは[ルーター側の機能](https://www.tp-link.com/jp/support/faq/1544/)を使っています。

### 外付けHDDの追加

Synology NASはUSBポートがあり、そこに外付けHDDを繋げればそれも共有対象にできます。外付けHDDの方はファイルシステムとしてNTFSも使えるので、既存のHDDをさして気軽に共有できます。

### Audio Station

音楽共有機能を使うことで家庭内で音楽を共有できます。自分専用のフォルダも用意できるので、共有側にはJ-POPを、自分専用側にはアニソンを入れておくことができます。なお、wmaには対応していないので、wmaファイルはmp3に変換する必要があります。

### Container Manager

NAS上でdockerコンテナを起動させることができます。パブリッククラウドに上げる前の動作確認に使えそうです。また、container registryを立ち上げることで、プライベートコンテナリポジトリを構築できます。