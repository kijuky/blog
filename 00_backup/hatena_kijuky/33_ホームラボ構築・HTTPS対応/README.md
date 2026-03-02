https://kijuky.hatenablog.com/entry/2026/02/25/205159

---

## やろうとしたこと

- 内部DNSと外部公開（Cloudflare Tunnel）でURLを統一したい
- https://hoge.fuga.com のような「覚えられるURL」で家庭内サービスへ安全にアクセスしたい
- Synology をリバースプロキシ兼TLS終端として運用

## 発生した不具合・課題

- Chromeで `ERR_CERT_COMMON_NAME_INVALID` が発生しHTTPS接続が警告扱いになる
- Let’s Encrypt証明書は取得できているのに解消しない
- 内部DNS・Cloudflare・リバースプロキシのどこが原因か切り分けが困難
- Synology GUIからの証明書取得はMAP-E環境のためHTTP検証に失敗
- acme.sh のDNS-01 challengeも最初は失敗（Cloudflare API設定ミス）
- さらに成功後も警告が残り、証明書割り当て済みにも関わらずエラー継続

## 解決方法

1. Cloudflare APIを利用して DNS-01 challenge（acme.sh）で証明書取得
2. ワイルドカード証明書を作成 fuga.com + *.fuga.com
3. Synologyへ「既存証明書のインポート」
4. DSM / リバースプロキシ / Webサービスへ証明書を再割当
5. NAS再起動

### 結果：

- 内部LAN
- 外部（Cloudflare Tunnel）

両方でHTTPS警告が解消し、同一URLでアクセス可能になった。

## なぜ解決できたか（理論）

1. HTTP検証が失敗した理由

   自宅回線は IPv6 MAP-E のため、外部からポート80へ到達できずLet’s EncryptのHTTP-01認証が成立しなかった。

   → DNS-01 challenge に切替で解決


2. 証明書エラーが消えなかった理由

   SynologyのnginxはTLSハンドシェイク時、

    1. SNI確定前にデフォルト証明書を提示
    2. その後にリバースプロキシで振り分け

   という動作をする。

   そのため*.fuga.com のみの証明書では初期提示証明書とホスト名が一致せずブラウザが拒否した。

   つまりブラウザは「アクセスしたドメイン」ではなく「最初に提示された証明書」を検証していた。

   → fuga.com をSANに含めたことで一致し解消

3. Cloudflareでは問題が出なかった理由

   外部アクセスでは

   ブラウザ → Cloudflare → 自宅

   となり、証明書はCloudflareが提示するためNAS側の証明書問題が隠れていた。

## 成果

- 内外どちらでも同一URLアクセスを実現
- 家庭内通信はローカル接続（トンネル回避）
- 安全なHTTPS通信確立
- リバースプロキシ基盤完成（サービス追加可能な状態）

## 学び

- HTTPSの検証は「どの証明書をサーバが最初に提示するか」で決まる
- ワイルドカード証明書はルートドメインを包含しない
- MAP-E環境ではDNS-01 challengeが実質必須
- リバースプロキシ運用では証明書設計がネットワーク設計と同じくらい重要

## 参考

https://masao-tec.com/automatic-certificate-renewal-using-acme-sh-dns-01-on-synology-nas-even-when-cloudflare-proxy-is-enabled/