https://qiita.com/kijuky/items/4ec852fbef5d3fad6a6a

---

GASはUrlFetchAppサービスを使うことで、外部のWebAPIとやり取りすることができます。ここではそれを使ってAWS S3にファイルをアップロードする方法を共有します。

# 結論

もうこのブログを引用したら終わりですが、つまりはそういうことです。

https://engetc.com/projects/amazon-s3-api-binding-for-google-apps-script/

サクッと下記スクリプトIDをライブラリに追加して、いい感じにアップロードします。

```
MB4837UymyETXyn8cv3fNXZc9ncYTrHL9
```

```javascript:コード.gs
var s3 = S3.getInstance(awsAccessKeyId, awsSecretKey);
var blob = UrlFetchApp.fetch("http://www.google.com").getBlob();
 
s3.putObject("bucket", "googlehome", blob, {logRequests:true});
 
//and to get it back
 var fromS3 = s3.getObject("bucket", "googlehome");
```

# 事例

ある案件で、他社から毎日メールで送られてくるcsvデータをグラフ表示する業務がありました。当初はcsvファイル名が安定していなかったりしていたので、手作業でメールの添付ファイルをダウンロードし、S3にDnD(ドラッグアンドドロップ)してアップロードをしていました。しばらくしてファイル名なども安定してきたので、この定型作業を自動化しようと考えました。メールはGMailを使っていたので、GMailを操作でき、かつ上述のライブラリでS3も操作できるので、GASを採用しました。

仕組みはうまく動き、もうメールから添付ファイルをダウンロード＆S3でアップロードの手作業はする必要はなくなりましたが、1つだけ問題がありました。それは送られてくるcsvファイルがShift-JISだったことです。手作業での操作ではファイルの変換作業などはしておらず、S3の先のシステム側でcsvがShift-JISを前提として処理していました。上記S3ライブラリでアップロードするとUTF-8になってしまうため、文字化けしてしまうのです。

ライブラリ側でアップロードするコンテンツをUTF-8に限定してしまっていたので、Shift-JISのファイルを上げるために、そこだけコメントアウトしてライブラリを使っています。

```javascript:S3.gs
...
/* puts an object into S3 bucket
 * 
 * @param {string} bucket 
 * @param {string} objectName name to uniquely identify object within bucket
 * @param {string} object byte sequence that is object's content
 * @param {Object} options optional parameters
 * @throws {Object} AwsError on failure
 * @return void
 */
S3.prototype.putObject = function (bucket, objectName, object, options) {
  options = options || {};

  var request = new S3Request(this);
  request.setHttpMethod('PUT');
  request.setBucket(bucket);
  request.setObjectName(objectName);
  
  var failedBlobDuckTest = !(typeof object.copyBlob == 'function' &&
                      typeof object.getDataAsString == 'function' &&
                      typeof object.getContentType == 'function'
                      );
  
  //wrap object in a Blob if it doesn't appear to be one
  if (failedBlobDuckTest) {
    object = Utilities.newBlob(JSON.stringify(object), "application/json");
    object.setName(objectName);
  }
  
  //request.setContent(object.getDataAsString()); // コメントアウト
  request.setContent(object); // 追加
  request.setContentType(object.getContentType());
  
  request.execute(options);  
};
```

```javascript:S3Request.gs
...
/* sets content of request
 * @param {string} content request content encoded as a string
 * @throws {string} message if invalid input
 * @return {S3Request} this request, for chaining
 */ 
S3Request.prototype.setContent = function(content) {
  //if (typeof content != 'string') throw 'content must be passed as a string' // コメントアウト
  this.content = content; 
  return this;
};
```

# まとめ

GASからS3にアップロードする方法を紹介しました。この機能は特にGSuite上のGASでは重要です。なぜなら、GSuite上のGASは必ずアカウント認証が必要なコンテンツですが、S3にファイルをアップロードできるということは、アカウント認証が必要ない場所にコンテンツを提供できる手段を得られる、ということです。GASが簡易デプロイシステムとして利用できることを意味します。同時にGSuiteのセキュリティの中で一番脆弱になる部分でもありますので、そこはリスクと天秤にかけつつ利用するとよいかと思います。

以上です。
