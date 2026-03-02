https://qiita.com/kijuky/items/be3408bc2e905a8aa238

---

ごきげんよう。みなさんは、写真は撮っていますか？それをGoogleフォトで管理していますか？スマホやPCでは特定のフォルダにある写真を自動でGoogleフォトに上げたりできますが、そうでない場合、手動でアップロードすることになると思います。そのような写真が定期的に発生する場合、何度も手動でアップロードしなくてはいけないので面倒ですよね。そこで今回は、GASを使ってGoogleフォトに画像をアップロードする処理を自動化する方法をご紹介します。

# はじめに

Googleフォトに対する処理を自動化する場合、[Photos Library API](https://developers.google.com/photos/library/reference/rest)を使用する必要があります。GAS用の公式ライブラリはないので、REST APIを使用する必要があります。加えて、このREST APIは[OAuth2.0認証](http://openid-foundation-japan.github.io/rfc6749.ja.html)を突破する必要があります。これらを利用するために[Google Cloud Platform](https://console.cloud.google.com/welcome)にプロジェクトを作成する必要があります。これらの初期設定が終われば、認証が通っている間はGoogleフォトに対する処理を自動化できます。

まずは上記を理解して、自動化すべきかを検討しましょう。あなたが、それでも自動化すべきと判断したなら、次の章から具体的にその実装に取り掛かりましょう。

# スプレッドシートに紐づくGASを作成

OAuth2.0認証のフローを進めるため、UIが利用できるGASを用意する必要があります。スプレッドシートに紐づくGASの場合、UIの操作が簡単なので、これを利用します。まずは[Googleドライブ](https://drive.google.com/drive/my-drive)からスプレッドシートを作成してください。

| ![image.png](./img/01.png) |
|:-:|
| **図1. 空白のスプレッドシートを作成**  |

次に、スプレッドシートに紐づくGASを作成します。

| ![image.png](./img/02.png) |
|:-:|
| **図2. GASを作成**  |

この時、後でスクリプトIDを使うので、控えておいてください。

| ![image.png](./img/03.png) | ![image.png](./img/04.png) |
|:-:|:-:|
| **図3. 「プロジェクトの設定」を選択** | **図4. 「スクリプトID」を控える** |

# GCPプロジェクトの作成と設定

では、Google Cloud Platform上に新規プロジェクトを作成し、Photos Library APIを使用できるようにする設定と、OAuth2.0認証用の設定を行います。まずはGoogleにログインした状態で[Google Cloud Platform](https://console.cloud.google.com/welcome)にアクセスします。

| ![image.png](./img/05.png) | ![image.png](./img/06.png) |
|:-:|:-:|
| **図5. 規約の同意画面** | **図6. プロジェクト選択画面を表示**  |

新規プロジェクトを作成します。

| ![image.png](./img/07.png) | ![image.png](./img/08.png) |
|:-:|:-:|
| **図7. 「新しいプロジェクト」を選択** | **図8. 新しいプロジェクトを作成** |

Photos Library API を有効化します。

| ![image.png](./img/09.png) | ![image.png](./img/10.png) | ![image.png](./img/11.png) |
|:-:|:-:|:-:|
| **図9. 新しいプロジェクトを選択** | **図10. ライブラリを表示** | **図11. photos library api を検索する**  |

| ![image.png](./img/12.png) | ![image.png](./img/13.png) |
|:-:|:-:|
| **図12. Photos Library API を選択する** | **図13. Photos Library API を有効にする**  |

OAuth 同意画面を作成します。

| ![image.png](./img/14.png) | ![image.png](./img/15.png) | ![image.png](./img/16.png) |
|:-:|:-:|:-:|
| **図14. 「認証情報」タブを選択** | **図15. 同意画面作成へ** | **図16. 組織タイプを選択** |

| ![image.png](./img/17.png) | ![image.png](./img/18.png) | ![image.png](./img/19.png) |
|:-:|:-:|:-:|
| **図17. アプリ情報1** | **図18. アプリ情報2** | **図19. スコープを追加** |

| ![image.png](./img/20.png) | ![image.png](./img/21.png) | ![image.png](./img/22.png) |
|:-:|:-:|:-:|
| **図20. スコープを検索** | **図21. スコープを選択** | **図22. スコープを決定** |

| ![image.png](./img/23.png) | ![image.png](./img/24.png) | ![image.png](./img/25.png) |
|:-:|:-:|:-:|
| **図23. テストユーザー画面をスキップ** | **図24. ダッシュボードに戻る** | **図25. 本番公開する** |

| ![image.png](./img/26.png) |
|:-:|
| **図26. 確認画面** |

OAuth クライアントを作成します。

| ![image.png](./img/27.png) | ![image.png](./img/28.png) | ![image.png](./img/29.png) |
|:-:|:-:|:-:|
| **図27. OAuth クライアント作成画面を表示** | **図28. 「ウェブ アプリケーション」を選択** | **図29. OAuth クライアントIDを作成** |

- 承認済みのJavaScript生成元 `https://script.google.com`
- 承認済みのリダイレクトURL `https://script.google.com/macros/d/スクリプトID/usercallback`

スクリプトIDは先ほど控えた値で置き換えてください

| ![image.png](./img/30.png) |
|:-:|
| **図30. クライアントIDとクライアントシークレットの表示** |

作成したクライアントIDとクライアントシークレットをメモしておいてください。

# プロジェクトとGASを紐づける

| ![image.png](./img/31.png) | ![image.png](./img/32.png) | ![image.png](./img/33.png) |
|:-:|:-:|:-:|
| **図31. プロジェクトの設定を開く** | **図32. 「プロジェクトを変更」を選択** | **図33. プロジェクト番号を入力し、「プロジェクトを設定」を選択** |

# GASでOAuth2.0認証する

公式が[apps-script-oauth2](https://github.com/googleworkspace/apps-script-oauth2)という、GAS で OAuth2.0 認証するライブラリを提供しているので、これを利用しましょう。このライブラリをGASに追加します。

| ![image.png](./img/34.png) | ![image.png](./img/35.png) | ![image.png](./img/36.png) |
|:-:|:-:|:-:|
| **図34. ライブラリを追加** | **図35. ライブラリを検索** | **図36. ライブラリを追加** |

入力するスクリプトID: `1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF`

次に、OAuth のスコープを追加します。

| ![image.png](./img/37.png) | ![image.png](./img/38.png) |
|:-:|:-:|
| **図37. 概要を開く** | **図38. 既存のスコープを控える** |

図では、既存のスコープは1つですが、複数ある場合は複数のURLを控えておいてください。

| ![image.png](./img/39.png) | ![image.png](./img/40.png) |
|:-:|:-:|
| **図39. プロジェクトの設定を開く** | **図40. 「appscript.json」を表示** |

| ![image.png](./img/41.png) | ![image.png](./img/42.png) |
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

| ![image.png](./img/43.png) | ![image.png](./img/44.png) | ![image.png](./img/45.png) |
|:-:|:-:|:-:|
| **図43. GASで追加したメニューを選択** | **図44. 認証ダイアログで「続行」を選択** | **図45. アカウントを選択** |

| ![image.png](./img/46.png) | ![image.png](./img/47.png) | ![image.png](./img/48.png) |
|:-:|:-:|:-:|
| **図46. 「詳細」を選択** | **図47. 「ページへ移動」を選択** | **図48. 「許可」を選択** |

次にGCPの認証を行います。

| ![image.png](./img/49.png) | ![image.png](./img/50.png) | ![image.png](./img/51.png) |
|:-:|:-:|:-:|
| **図49. GASで追加したメニューを選択** | **図50. 「アクセス認証」を選択** | **図51. アカウントを選択** |

| ![image.png](./img/52.png) | ![image.png](./img/53.png) | ![image.png](./img/54.png) |
|:-:|:-:|:-:|
| **図52. 「詳細」を選択** | **図53. 「ページへ移動」を選択** |  **図54. 「許可」を選択** |

| ![image.png](./img/55.png) | ![image.png](./img/56.png) | ![image.png](./img/57.png) |
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

