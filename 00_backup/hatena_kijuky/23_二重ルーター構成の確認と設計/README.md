https://kijuky.hatenablog.com/entry/2026/02/20/221300

---

# やりたかったこと

- [NSD-G1000T](https://www.nuro.jp/pdf/device/manual_NSD-G1000T.pdf)と[AXE5400](https://www.tp-link.com/jp/home-networking/wifi-router/archer-axe5400/)を二重ルーター構成で運用し、Wi-Fi安定性を確保しつつ、IPアドレスやDHCP範囲を整理して、将来的に[UDR7](https://jp.store.ui.com/jp/ja/products/udr7)導入も考慮したネットワーク設計を行う。

https://internet.watch.impress.co.jp/docs/review/2081890.html

# 発生した課題

- NSD再起動が必要なWi-Fi切断問題
- [DMZ](https://ja.wikipedia.org/wiki/%E9%9D%9E%E6%AD%A6%E8%A3%85%E5%9C%B0%E5%B8%AF_(%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3\))設定が[MAP-E](https://ja.wikipedia.org/wiki/IPv6%E7%A7%BB%E8%A1%8C%E6%8A%80%E8%A1%93#MAP)環境下で利用不可
- DHCP範囲と固定IPの割り当てが混乱していた
- ルーター間ネットワーク（192.168.0.x と 192.168.10.x）の管理が複雑

# 課題の解決方法

- NSDはルーターのまま残し、AXEをメインのWi-Fi機器として運用
- DMZ利用は諦め、IPアドレス管理は小さい数字を固定、DHCPは大きい番号に設定
- 二重ルーター間のネットワークを整理し、AXE LAN 192.168.10.x、NSD LAN 192.168.0.xで統一

# 解決方法の理論的説明

- 二重ルーターではWAN側/LAN側で異なるネットワークを割り当てることで、DHCPや固定IPの衝突を避ける
- 小さい番号を手動固定することで、DHCPとIP重複を防ぎ、IP管理が容易になる
- DMZはMAP-E下で無効のため、内部ネットワークからのルーティングで代替する方針に変更