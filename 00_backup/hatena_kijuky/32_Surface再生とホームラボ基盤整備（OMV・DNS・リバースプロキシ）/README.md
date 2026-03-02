https://kijuky.hatenablog.com/entry/2026/02/25/140924

---

## やろうとしたこと / 目的

- Surface Pro + OpenMediaVault を常時稼働サーバとして利用
- 無線LAN接続で設置場所の自由度を上げる
- Docker コンテナ（Dashy等）の運用開始
- 家庭内向け名前解決（home.arpa）の構築
- LAN内サービスをドメイン名でアクセス可能にする

## 発生した課題・問題

### Surface / OS 周り

- 無線LANドライバが認識されない
- 画面消灯すると Wi-Fi も停止してサーバが切断される
- ログイン情報を忘れてログイン不能

### Docker / アプリ

- Dashy コンテナが起動失敗
- not a directory マウントエラー
- Dashy が頻繁にクラッシュ
- ERR_HTTP_HEADERS_SENT

### DNS / ネットワーク

- 名前解決が不安定
- dig では解決するが ping では解決しない
- Mac のスリープ復帰後に接続失敗
- Synology DNS が外向き名前解決できていない
- デュアルNIC（メインLAN + 隔離LAN）で誤ったIPが返る
- immich.home.arpa → 192.168.10.21
- 隔離側からは 192.168.20.21 に居るため到達不可

⸻

## 解決方法

### Surface

- rootログインから設定修正
- サスペンド（スリープ）を無効化
- 画面消灯のみ有効に設定

→ Wi-Fi接続を維持したまま省電力化

### nano

- Ctrl + O 保存
- Ctrl + X 終了

### Docker / Dashy

- ホスト側に 実ファイルを作成してから bind mount
- conf.yml を正しく配置
- Dashy設定修正によりクラッシュ解消

### DNS

- Synology DNS Server に home.arpa のプライマリゾーン作成
- DHCPでDNS配布をSynologyに統一
- DNSフォワーダを有効化
- フォワーダ方式を「先にフォワード」に設定

### 名前解決不安定の対処

#### 原因：

- 内向きDNSのみ解決
- 外向きDNS解決不可
- Macがキャッシュした失敗を保持

#### 対策：

- Synology DNSで外部DNSへフォワード設定

→ 内外両方の名前解決が可能になり安定

### デュアルネットワーク問題

- Synologyが2つのIPを持つためAレコードが片側IPになる問題発生
- Split horizon DNS 相当の問題と判明
- 完全な自動解決は困難なため、必要に応じIP直打ち併用の方針

## 理論的説明

### サスペンドと画面消灯

LinuxのサスペンドはCPU・NIC・USBバスへの給電が停止するため、Wi-Fiインターフェースも停止する。画面消灯は単なるDPMSでありネットワークは維持される。

### digとpingの差

- dig → DNSサーバへ直接問い合わせ
- ping → OSの名前解決キャッシュを使用

このためDNSが修正されても、OSキャッシュが残っている間は失敗する。

### DNSフォワーダ

ローカルゾーン（home.arpa）以外の問い合わせを上流DNSへ転送することで

- 内部サービス名
- 外部インターネット

の両方を1台のDNSで解決できる。

### デュアルNIC問題

1台のホストに複数ネットワークがある場合、DNSは「接続元ネットワーク」を判断できないため不適切なIPを返すことがある。これは企業ネットワークでいう Split DNS 問題に相当する。

## 現在の状態

- Surface OMV サーバ常時稼働
- Wi-Fi接続維持（画面消灯OK）
- Docker運用開始
- Dashy 安定稼働
- home.arpa による家庭内名前解決成功
- LAN内サービスをドメイン名でアクセス可能
- DNS安定化完了