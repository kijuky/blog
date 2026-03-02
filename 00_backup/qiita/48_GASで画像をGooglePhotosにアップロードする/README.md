https://qiita.com/kijuky/items/be3408bc2e905a8aa238

---

ごきげんよう。みなさんは、写真は撮っていますか？それをGoogleフォトで管理していますか？スマホやPCでは特定のフォルダにある写真を自動でGoogleフォトに上げたりできますが、そうでない場合、手動でアップロードすることになると思います。そのような写真が定期的に発生する場合、何度も手動でアップロードしなくてはいけないので面倒ですよね。そこで今回は、GASを使ってGoogleフォトに画像をアップロードする処理を自動化する方法をご紹介します。

# はじめに

Googleフォトに対する処理を自動化する場合、[Photos Library API](https://developers.google.com/photos/library/reference/rest)を使用する必要があります。GAS用の公式ライブラリはないので、REST APIを使用する必要があります。加えて、このREST APIは[OAuth2.0認証](http://openid-foundation-japan.github.io/rfc6749.ja.html)を突破する必要があります。これらを利用するために[Google Cloud Platform](https://console.cloud.google.com/welcome)にプロジェクトを作成する必要があります。これらの初期設定が終われば、認証が通っている間はGoogleフォトに対する処理を自動化できます。

まずは上記を理解して、自動化すべきかを検討しましょう。あなたが、それでも自動化すべきと判断したなら、次の章から具体的にその実装に取り掛かりましょう。

# スプレッドシートに紐づくGASを作成

OAuth2.0認証のフローを進めるため、UIが利用できるGASを用意する必要があります。スプレッドシートに紐づくGASの場合、UIの操作が簡単なので、これを利用します。まずは[Googleドライブ](https://drive.google.com/drive/my-drive)からスプレッドシートを作成してください。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/2a27ac58-a5dc-45f7-9651-631b63b73c8b.png) |
|:-:|
| **図1. 空白のスプレッドシートを作成**  |

次に、スプレッドシートに紐づくGASを作成します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/b0de6293-793f-3d39-e8e4-d6c2c71e0f57.png) |
|:-:|
| **図2. GASを作成**  |

この時、後でスクリプトIDを使うので、控えておいてください。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/7be06140-3688-f7ef-f4e2-90d4c6115443.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/3967d53b-6f8e-371e-abb6-07e4ff838621.png) |
|:-:|:-:|
| **図3. 「プロジェクトの設定」を選択** | **図4. 「スクリプトID」を控える** |

# GCPプロジェクトの作成と設定

では、Google Cloud Platform上に新規プロジェクトを作成し、Photos Library APIを使用できるようにする設定と、OAuth2.0認証用の設定を行います。まずはGoogleにログインした状態で[Google Cloud Platform](https://console.cloud.google.com/welcome)にアクセスします。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/4cdbae4b-98a9-a872-4636-7cbf8c3f251b.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/e8f3a632-0103-b500-1215-a3554d05a451.png) |
|:-:|:-:|
| **図5. 規約の同意画面** | **図6. プロジェクト選択画面を表示**  |

新規プロジェクトを作成します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/a38f51c8-c4d4-61a9-8d44-25d7e408ca5e.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/0b4ca0d9-0b79-db13-6e97-8e7e9f601f11.png) |
|:-:|:-:|
| **図7. 「新しいプロジェクト」を選択** | **図8. 新しいプロジェクトを作成** |

Photos Library API を有効化します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/bb8cf042-aa60-74c5-66f0-e85d0b9f5d34.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/46772631-7d66-f7af-0de8-0f8eef92f617.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/0bec16f9-5159-0a74-2f63-8e55ee533d89.png) |
|:-:|:-:|:-:|
| **図9. 新しいプロジェクトを選択** | **図10. ライブラリを表示** | **図11. photos library api を検索する**  |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/9da156cb-ce2c-9d6d-26ef-3f77ec922796.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/2a456334-ee6a-0daf-27b9-ff000077807a.png) |
|:-:|:-:|
| **図12. Photos Library API を選択する** | **図13. Photos Library API を有効にする**  |

OAuth 同意画面を作成します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/3e0d260c-322a-7df9-c4b4-e8f56d35b256.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/6b07f9d7-f0a8-78e5-12f3-0c6c8ff154c7.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/4cbe25d7-2576-77c5-8859-80226a44a510.png) |
|:-:|:-:|:-:|
| **図14. 「認証情報」タブを選択** | **図15. 同意画面作成へ** | **図16. 組織タイプを選択** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/91dfc607-d23f-b6c8-b068-dfeabb2392db.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/754335d6-a145-8ad6-2ed0-461ca5dc47c8.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/8ef96abf-e850-85fe-2d2b-0c7b6682ff5a.png) |
|:-:|:-:|:-:|
| **図17. アプリ情報1** | **図18. アプリ情報2** | **図19. スコープを追加** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/9963af20-73f1-915e-53af-bd4366c71d2d.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/42eb5505-9f9f-3209-8d48-824313e3711d.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/3acf6dde-105a-085c-78d8-5b3e2847e973.png) |
|:-:|:-:|:-:|
| **図20. スコープを検索** | **図21. スコープを選択** | **図22. スコープを決定** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/d9b6bec3-992d-1001-8bfa-d7ecfa5e0c83.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/83802776-5897-8c0d-5051-1d6611251a72.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/1734bd18-7348-b9a5-b576-e230e62cbb3c.png) |
|:-:|:-:|:-:|
| **図23. テストユーザー画面をスキップ** | **図24. ダッシュボードに戻る** | **図25. 本番公開する** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/b49a302a-901e-b3dc-e603-f3b657cc4959.png) |
|:-:|
| **図26. 確認画面** |

OAuth クライアントを作成します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/d237f861-1943-c9f4-d227-c12883e0ae2e.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/94948a46-b427-347c-862c-b8537fbffd9a.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/a78f26e7-7b4d-d025-bbe5-ebfa32f9ba6e.png) |
|:-:|:-:|:-:|
| **図27. OAuth クライアント作成画面を表示** | **図28. 「ウェブ アプリケーション」を選択** | **図29. OAuth クライアントIDを作成** |

- 承認済みのJavaScript生成元 `https://script.google.com`
- 承認済みのリダイレクトURL `https://script.google.com/macros/d/スクリプトID/usercallback`

スクリプトIDは先ほど控えた値で置き換えてください

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/30fe0e65-9e33-256b-c070-91c45228f1ac.png) |
|:-:|
| **図30. クライアントIDとクライアントシークレットの表示** |

作成したクライアントIDとクライアントシークレットをメモしておいてください。

# プロジェクトとGASを紐づける

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/6780de7c-9478-2f68-e233-b5121f0deec3.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/f213b002-5c44-a83c-45d2-c61737d833b3.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/c4de9aeb-7205-89da-039b-5d430a5a6030.png) |
|:-:|:-:|:-:|
| **図31. プロジェクトの設定を開く** | **図32. 「プロジェクトを変更」を選択** | **図33. プロジェクト番号を入力し、「プロジェクトを設定」を選択** |

# GASでOAuth2.0認証する

公式が[apps-script-oauth2](https://github.com/googleworkspace/apps-script-oauth2)という、GAS で OAuth2.0 認証するライブラリを提供しているので、これを利用しましょう。このライブラリをGASに追加します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/64989109-29b8-1e26-0202-f8fec670145a.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/a6d49306-b755-527b-56ed-b49869b6933d.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/0136fa7f-13a4-2dc0-a3a0-254bc5499990.png) |
|:-:|:-:|:-:|
| **図34. ライブラリを追加** | **図35. ライブラリを検索** | **図36. ライブラリを追加** |

入力するスクリプトID: `1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF`

次に、OAuth のスコープを追加します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/2546b5c4-6785-9d51-fe75-5e8d8c6c4942.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/074085f4-c717-1181-2352-47b2b314ffe6.png) |
|:-:|:-:|
| **図37. 概要を開く** | **図38. 既存のスコープを控える** |

図では、既存のスコープは1つですが、複数ある場合は複数のURLを控えておいてください。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/42636726-a6b7-3570-f5e6-3b20e880cb43.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/2ce3b13e-1025-bf2c-9b82-8a0bad31479e.png) |
|:-:|:-:|
| **図39. プロジェクトの設定を開く** | **図40. 「appscript.json」を表示** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/3b5470b1-941c-e7e4-062b-67839a9dada4.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/366eef41-eddf-3770-2b84-35182c485923.png) |
|:-:|:-:|
| **図41. エディタを開く** | **図42. スコープを追加** |

スコープは、下記を追加しました。

```json:appscript.json
  "oauthScopes": [
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/userinfo.email",
    "https://www.googleapis.com/auth/script.container.ui",
    "https://www.googleapis.com/auth/photoslibrary.appendonly"
  ]
```

「コード」スクリプトを作成し、次のコードを貼り付けます。

```javascript:コード.gs
function onOpen() {
  SpreadsheetApp
    .getUi()
    .createMenu(`スクリプト`)
    .addItem(`GooglePhoto OAuth2.0認証`, `refreshAuth`)
    .addToUi();
}

function refreshAuth() {
  const ui = SpreadsheetApp.getUi();
  const service = checkOAuth();
  if (!service.hasAccess()) {
    console.log(`no access`);
    const output = HtmlService.createHtmlOutputFromFile(`template`).setHeight(310).setWidth(500).append(authpage()).setSandboxMode(HtmlService.SandboxMode.IFRAME);
    ui.showModalDialog(output, `OAuth2.0認証`);
  } else {
    console.log(`service success.`);
  }
}

function authpage() {
  const service = checkOAuth();
  const authorizationUrl = service.getAuthorizationUrl();
  return `<center><b><a href='${authorizationUrl}' target='_blank' onclick='closeMe();'>アクセス承認</a></b></center>`;
}

function checkOAuth() {
  return OAuth2.createService(`PhotosAPI`)
    .setAuthorizationBaseUrl(`https://accounts.google.com/o/oauth2/auth`)
    .setTokenUrl(`https://accounts.google.com/o/oauth2/token`)
    .setClientId(`クライアントID`)
    .setClientSecret(`クライアントシークレット`)
    .setCallbackFunction(`authCallback`)
    .setPropertyStore(PropertiesService.getScriptProperties())
    .setScope(`https://www.googleapis.com/auth/photoslibrary.appendonly`)
    .setParam(`login_hint`, Session.getActiveUser().getEmail())
    .setParam(`access_type`, `offline`)
    .setParam(`approval_prompt`, `force`);
}

function authCallback(request) {
  const service = checkOAuth();
  const isAuthorized = service.handleCallback(request);
  if (isAuthorized) {
    return HtmlService.createHtmlOutput(`認証に成功しました。ページを閉じて再度実行ボタンを押してください`);
  } else {
    return HtmlService.createHtmlOutput(`認証に失敗しました。再度お試しください`);
  }
}
```

「template」HTMLを作成し、次のコードを貼り付けます。

```html:template.html
<!DOCTYPE html>
<html>
  <head>
    <style>
      center {
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
      }
      center > b > a {
        text-decoration: none;
        color: aliceblue;
        background-color: cadetblue;
        padding: 30px;
        border-radius: 15px;
      }
    </style>
    <base target="_top">
  </head>
  <body style="margin: 0;">
  </body>
</html>
```

それではOAuth認証をしてアクセストークンを取得しましょう。OAuth認証は計2回行う必要があります。1つ目はGASの認証で、もう1つはGCPの認証です。

ではGASの認証から行います。スプレッドシートを更新して、「スクリプト」から「GooglePhoto OAuth2.0認証」を選択します。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/741746b7-6da2-dba0-f7e8-3c674cae620f.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/c2e17f0f-0ef8-7302-afa9-217a411e10f6.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/abb7d154-f515-cb79-8bea-dade9f93f669.png) |
|:-:|:-:|:-:|
| **図43. GASで追加したメニューを選択** | **図44. 認証ダイアログで「続行」を選択** | **図45. アカウントを選択** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/b74ad57a-224d-3a2e-1404-f7d1f7e7dfeb.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/1e32db76-7a11-9a75-6f2a-9d0334e1ce8c.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/97b39adb-fe79-6f04-9d83-869d1223600f.png) |
|:-:|:-:|:-:|
| **図46. 「詳細」を選択** | **図47. 「ページへ移動」を選択** | **図48. 「許可」を選択** |

次にGCPの認証を行います。

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/741746b7-6da2-dba0-f7e8-3c674cae620f.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/718fd483-61e8-6e71-9187-73d11688b549.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/3a0ffce2-7a2a-dcf1-304d-0a814a3ae3d5.png) |
|:-:|:-:|:-:|
| **図49. GASで追加したメニューを選択** | **図50. 「アクセス認証」を選択** | **図51. アカウントを選択** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/48972b97-6089-5d93-4845-ccd93d7ceaf9.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/2dfa6d28-dbfb-d515-e502-f14f1c49985e.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/c6b97e9e-af4f-a3d2-38e4-a7f50aed2937.png) |
|:-:|:-:|:-:|
| **図52. 「詳細」を選択** | **図53. 「ページへ移動」を選択** |  **図54. 「許可」を選択** |

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/b55c3219-1e76-7d91-10b1-06ffa02aaa13.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/42636726-a6b7-3570-f5e6-3b20e880cb43.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/d5f78a29-75a9-27c2-82de-de5eb9b17953.png) |
|:-:|:-:|:-:|
| **図55. 認証成功画面** | **図56. GASのプロジェクト設定を開く** | **図57. アクセストークンを確認** |

認証ができると、スクリプトプロパティにアクセストークンが設定されます。

次章で、このアクセストークンを使ってPhotos Library APIを叩いてみましょう。

# Photos Library API を使った画像のアップロード

画像のアップロードは2つのAPIを叩く必要があります。1つは画像そのものをアップロードするAPI、もう1つはその画像をGoogleフォトに登録するAPIです。今回は、アップロードしたい画像はURLからダウンロードできるものとします（最終的にはBlobが取得できるなら何でも良いです）。それでは、次のコードを貼り付けます。

```javascript:アップロード.gs
function uploadToGooglePhotos(urls, sessionCookie) {
  // 画像を取得
  const blobs = urls
    .map(url => UrlFetchApp
      .fetch(
        url,
        {
          method: `GET`,
          headers: {
            Cookie: sessionCookie // 画像の取得に認証が必要な場合
          }
        }
      )
      .getBlob()
    );

  // 認証
  const accessToken = checkOAuth().getAccessToken();

  // アップロードトークンを取得
  const uploadTokens = blobs
    .map(blob => UrlFetchApp
      .fetch(
        `https://photoslibrary.googleapis.com/v1/uploads`,
        {
          method: `POST`,
          headers: {
            Authorization: `Bearer ${accessToken}`,
            "X-Goog-Upload-Content-Type": `image/jpeg`, // 画像によって変える。今回は jpg 固定とする。
            "X-Goog-Upload-Protocol": `raw`
          },
          contentType: `application/octet-stream`,
          payload: blob,
          muteHttpExceptions: true
        }
      )
      .getContentText()
    );

  // 画像を登録
  const chunk = (arr, size) => arr
    .reduce(
      (newarr, _, i) => (i % size ? newarr : [...newarr, arr.slice(i, i + size)]),
      []
    );
  chunk(uploadTokens.map((uploadToken, index) => [uploadToken, blobs[index].getName()]), 50)
    .forEach(uploadTokenAndFileNames => UrlFetchApp
      .fetch(
        `https://photoslibrary.googleapis.com/v1/mediaItems:batchCreate`,
        {
          method: `POST`,
          headers: {
            Authorization: `Bearer ${accessToken}`
          },
          contentType: `application/json`,
          payload: JSON.stringify({
            newMediaItems: uploadTokenAndFileNames
              .map(([uploadToken, fileName]) => ({
                description: ``,
                simpleMediaItem: {
                  fileName: fileName,
                  uploadToken: uploadToken
                }
              })
            )
          }),
          muteHttpExceptions: true
        }
      )
    );
}
```

`mediaItems:batchCreate` は1度に50枚の画像の登録ができるので、`chunk` 関数を使って、与えられた画像を50枚ずつに分けてアクセスしています。

実際に使う場合は、こんな感じで使います。

```javascript
  const urls = ["https://~~~~.jpg", "https://~~~~.jpg", ...]; // 画像URLの配列
  const sessionCookie = "XXX=YYY;XXX=YYY;..."; // 画像の取得に認証が必要な場合
  uploadToGooglePhotos(urls, sessionCookie);
```

[Googleフォト](https://photos.google.com/)を開いてみましょう。画像がアップロードされたことが確認できましたか？おめでとうございます :tada:

# まとめ

- Photos Library API を使って GAS で Googleフォトに画像を追加しました。
    - OAuth 認証を突破するために GCP に新規プロジェクトを作成、OAuth クライアントIDを作成しました。
    - クライアントIDを作成するまでの流れは、OAuthのワークフローを理解していないと、「今一体何の作業をしているんだっけ？」と迷子になるくらいには手順がたくさん必要で、なかなかしんどい。
- 図をたくさん貼ったけど、Google様の機嫌次第でこの辺のUIサクッと変わるから、スクショはすぐ役に立たなくなりそう...
- みなさんも自動化トライして、日常の定型作業から解放されましょう！

# 参考

https://qiita.com/hiteto/items/f63584045df31dff31d2

