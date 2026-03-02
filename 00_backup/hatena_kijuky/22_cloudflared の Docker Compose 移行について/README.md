https://kijuky.hatenablog.com/entry/2026/02/20/215243

---

# やろうとしたこと、やりたかったこと

- [Immich](https://immich.app/) サーバーを外部公開するため、[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/) を利用
- 以前は `docker run` で cloudflared を単体起動していたが、運用を安定・自動化するために docker-compose に組み込みたい
- 目標：
    - Cloudflare Tunnel を compose 管理下で常駐させる
    - Immich と cloudflared の通信を安定させ、Synology 環境でも i/o timeout を回避する

# 発生した不具合、課題、問題

1. docker-compose 化後に cloudflared から Immich への接続がタイムアウト

    ```console
    ERR Request failed error="Unable to reach the origin service"
    ```

    - LAN からは Immich に接続できる
    - cloudflared コンテナからホストLAN経由で接続するとタイムアウトになる

2. default bridge への接続で network-scoped alias が使えない

    ```console
    network-scoped alias is supported only for containers in user-defined networks
    ```

    - default bridge ではコンテナ名参照や aliases が使えない

# その不具合の解消方法、解決策

- Cloudflared を Immich サービスの Compose に組み込む
    - Immich サービスと同じ user-defined network に接続
    - cloudflared 側の宛先を `immich_server:2283` のようにコンテナ名で指定
- Docker Compose 内でネットワークを定義する例：

    ```yaml
    networks:
      immich_net:
        driver: bridge

    services:
      immich_server:
        networks:
          - immich_net
      cloudflared:
        image: cloudflare/cloudflared:latest
        command: tunnel --no-autoupdate run --token ${CLOUDFLARED_TOKEN}
        networks:
          - immich_net
    ```

    - こうすることで、Synology のファイアウォールや LAN 経路に依存せず安定した接続が可能

# なぜそれで解決できたのか、理論的な説明

1. コンテナ間通信を LAN 経由ではなく Docker 内部ネットワークで完結
- `docker run` の default bridge では、Synology の FW/NAT によって LAN 方向の通信がタイムアウトすることがあった
- user-defined network にすることで、Docker が提供する内部DNSを使った 直接コンテナ名参照 が可能になり、タイムアウトが解消された
2. network-scoped alias を使える
- default bridge では aliases が使えない
- user-defined network 上では service 名・aliases が使えるため、cloudflared から immich_server で通信可能
3. Compose 管理下に置くことで自動復帰・永続化
- Synology 再起動時もネットワーク接続が保持される
- cloudflared と Immich を同じ Compose にまとめることで、運用・更新・再起動が簡単になる

# 💡 結論

- cloudflared を安定して Immich に接続させたい場合は SaaS 側（Immich）の Docker Compose に組み込むのが最適
- 外部 default bridge や単体コンテナとして動かす構成は、Synology 環境ではタイムアウトや aliases の制限により運用が不安定になりやすい
