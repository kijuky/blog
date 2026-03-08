いろいろチャッピーと壁打ちした結果、今あるハードウェアを活かしてこんな感じにしようと思う。

- Lenovo M720q
  - PVEが載ってるマシン。
  - CPUはi9 9900T、RAMは64GBに換装済み。
  - NVMe SSD 2TB にPVE本体、ISOイメージ、軽めのVMイメージ。
  - SATA SSD 2TB にVMイメージ。
    - 将来的にはここを4TBにしたい。
- Synology
  - ここにVMのバックアップをとる。

これとは別に新マシンとして下記を検討中。

<table>
  <tr>
    <th>カテゴリ</th>
    <th>製品</th>
    <th>値段</th>
    <th>割合</th>
    <th>備考</th>
  </tr>
  <tr>
    <th>CPU</th>
    <td><a href="https://www.tradeinn.com/techinn/ja/amd-r9-7900-3.7ghz-%E3%82%B7%E3%83%BC%E3%83%94%E3%83%BC%E3%83%A6%E3%83%BC/139671816/p?id_producte=19047818&country=jp&utm_source=chatgpt.com">AMD Ryzen 9 7900</a></td>
    <td>52,859 円</td>
    <td>11.9 %</td>
    <td>安いやつだと<a href="https://www.pc-koubou.jp/products/detail.php?product_id=1152676&utm_source=chatgpt.com">AMD Ryzen 9 7900X</a> 44,082円</td>
  </tr>
  <tr>
    <th>M/B</th>
    <td><a href="https://www.janpara.co.jp/sale/search/detail/?SRCODE=39253662">MSI MAG B650M MORTAR WIFI</a></td>
    <td>13,980 円</td>
    <td>3.2 %</td>
    <td></td>
  </tr>
  <tr>
    <th>RAM</th>
    <td><a href="https://amazon.co.jp/-/en/Kingston-2x32GB-6400MT-Desktop-Gaming/dp/B0CYM412PY?utm_source=chatgpt.com">Kingston FURY Beast DDR5 64GB</a></td>
    <td>199,334 円</td>
    <td>45.0 %</td>
    <td></td>
  </tr>
  <tr>
    <th>NVMe</th>
    <td><a href="https://www.amazon.co.jp/%E3%82%A6%E3%82%A8%E3%82%B9%E3%82%BF%E3%83%B3%E3%83%87%E3%82%B8%E3%82%BF%E3%83%AB-WDS200T3B0E-SN580-SSD%EF%BC%882TB-900TBW/dp/B0C8WPRM9T?source=ps-sl-shoppingads-lpcontext&ref_=fplfs&smid=A1M8RH7PAVVIJI&utm_source=chatgpt.com&th=1">WD Blue SN580 2TB NVMe SSD</a></td>
    <td>59,799 円</td>
    <td>13.5 %</td>
    <td>これはLenovo M720qを2TB->512GBに換装して、余った2TBをこっちに乗せても良さそう。</td>
  </tr>
  <tr>
    <th>SATA</th>
    <td><a href="https://store.shopping.yahoo.co.jp/zappinya/287019916611.html?sc_e=slga_fpla&utm_source=chatgpt.com">Crucial MX500 4TB SATA SSD</a></td>
    <td>79,800 円</td>
    <td>18 %</td>
    <td></td>
  </tr>
  <tr>
    <th>PSU</th>
    <td><a href="https://joshinweb.jp/peripheral/74740/4537694358330.html?utm_source=chatgpt.com">Corsair RM750e 750W</a></td>
    <td>11,980 円</td>
    <td>2.7 %</td>
    <td></td>
  </tr>
  <tr>
    <th>Cooler</th>
    <td><a href="https://www.pc-koubou.jp/products/detail.php?product_id=814760&utm_source=chatgpt.com">Noctua NH‑U12S redux</a></td>
    <td>6,800 円</td>
    <td>1.5 %</td>
    <td></td>
  </tr>
  <tr>
    <th>Case</th>
    <td><a href="https://www.amazon.co.jp/Fractal-Design-%E3%83%9F%E3%83%89%E3%83%AB%E3%82%BF%E3%83%AF%E3%83%BCPC%E3%82%B1%E3%83%BC%E3%82%B9-%E5%BC%B7%E5%8C%96%E3%82%AC%E3%83%A9%E3%82%B9%E3%83%A2%E3%83%87%E3%83%AB-FD-C-DEF7C-03/dp/B0892FJR9P?source=ps-sl-shoppingads-lpcontext&ref_=fplfs&utm_source=chatgpt.com&th=1">Fractal Design Define 7 Compact</a></td>
    <td>18,131 円</td>
    <td>4.1 %</td>
    <td></td>
  </tr>
  <tfoot>
    <tr>
      <th>合計</th>
      <td></td>
      <td>442,683 円</td>
      <td>100 %</td>
      <td></td>
    </tr>
  </tfoot>
</table>

あれ...? チャッピーに予算20万って言ったのに、全然予算オーバーしているじゃん、なんだこれ？

一応これだと、今後の拡張として

- RAMを増やしてVMたくさん載せられる
- GPUを生やしてローカルLLMできる

って言われたけど、どうなんだろ？
