https://kijuky.hatenablog.com/entry/2021/06/08/185423

----

はてなブログを開設して記事を書いてみたところ、記事入力画面がプロポーショナルフォントでソースコードが書きづらかったので、記事入力画面を等幅フォント（Courier New）にするブックマークレットを作りました。

[はてなブログの記事編集ページのフォントを Courier New (等幅フォント)にする](http://let.hatelabo.jp/kijuky/let/i-mCgve-gcAA)

（iframeって表示されるのかな...？）

## 作り方

まずは Developer ツールでセレクタとプロパティを調べつつ、動くコードを書きます。

document.getElementById("body").style.fontFamily="Courier New"

https://developer.mozilla.org/ja/docs/Web/CSS/CSS_Properties_Reference

等幅フォント一覧はこちらとかを参照しました。

https://www.bugbugnow.net/2020/02/font-family-mono.html#toc-12

うん、良さそう。

次に Bookmarklet に仕立て上げます。 `javascript:(function(){` と `})()` で囲んであげます。

https://qiita.com/xtetsuji/items/e8b61bb39c41b7a9345e

色々とぐぐってみたらはてなレットというサイトを発見。せっかくなので登録してみました（上のリンク）。

作成する際にはてラボ人間性センターというスパムチェッカーがありまして、こちらキン肉マンやら落語やらの知識がなかったので１０回くらい失敗しました...今回ので一番時間かかったかも。