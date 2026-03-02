https://qiita.com/kijuky/items/58a6794095f29fedc16e

---

皆さん、リモートワーク捗っていますか？この記事では、現在アクティブなSlackチャンネルの内容を発話させて物理オフィスの賑やかなイメージを取り戻すことができます。

# 使う技術の紹介

## Web Speech API

ブラウザ上で文字から音声を再生することができるAPIです。意外と多くのブラウザがサポートしていてびっくり。

- https://qiita.com/hmmrjn/items/be29c62ba4e4a02d305c

## Mutation Observer

DOM が変わったときに通知を受ける事ができる機能です。Slackのアクティブなチャンネルに新たな投稿があったのをフックするために使います。

- https://ja.javascript.info/mutation-observer

ちょっと手こずったのは、子孫の要素の変更を受け取るためには子の要素の変更を受け取らないといけないこと。つまり、`subtree:true` だけではダメで、`childList:true`も指定する必要があります。

# 発話させよう

この２つが揃ったあとは

- DOMの変更を受け取って
- テキストを発話させる

と良いです。というわけで、サクッと下記を Chrome の開発者ツールのコンソールに突っ込むと喋りだします。

```javascript
const elem = document.querySelectorAll(".c-virtual_list__scroll_container")[1];
const observer = new MutationObserver(records => {
  const record = records[records.length - 1];
  if (record.nextSibling) return;
  record.addedNodes.forEach(node => {
    if (node.ariaExpanded !== "false") return;
    const matchResult = /:\d\d(.+)/gs.exec(node.textContent);
    if (matchResult) {
      const text = matchResult[1];
      speechSynthesis.speak(new SpeechSynthesisUtterance(text));
    }
  });
});
const config = {subtree: true, childList: true, characterData: true};
observer.observe(elem, config);
//observer.disconnect(); // 発話を終了する
```

普段からコピペするのも大変なので、ブックマークレットにすると便利でしょう。下記ブックマークレットでは、再度実行すると「現在の発話を中断し」「Observerを切断し」ます。何かおかしくなったら再度実行してください。

https://gist.github.com/kijuky/053c7988684f55bf7f0e7c3ddb50cf9b

# やってみた感想

- Botがいるチャンネルとかだと微妙かも。
- チャンネルを変更すると、そのチャンネルの最後の発言が発話される。
- ~~**[追記] 過去の発言を表示するとそれを延々と喋り続けてしまう。**~~ -> 修正した
- 3日くらいは楽しめそう。
