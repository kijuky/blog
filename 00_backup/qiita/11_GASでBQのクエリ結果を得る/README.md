https://qiita.com/kijuky/items/abb6091f0913d90454a1

---

業務でGASとBQ(BigQuery)を連携して、BQのクエリ結果をGoogleスプレッドシートに転記する、みたいなことが実現できました。この記事ではクエリ結果を得る部分について共有したいと思います。

# なにはともあれコード例
大元の出典があるのですが、そのコードを1つの関数にまとめて、 `var` を `let` や `const` に書き換えました。

```javascript
function queryTo(query, projectId) {
  let queryResults = BigQuery.Jobs.query({query: query}, projectId);
  const jobId = queryResults.jobReference.jobId;
  
  // Check on status of the Query Job. Wait until job status become complete.
  let sleepTimeMs = 500;
  while (!queryResults.jobComplete) {
    Utilities.sleep(sleepTimeMs);
    sleepTimeMs *= 2;
    queryResults = BigQuery.Jobs.getQueryResults(projectId, jobId);
  }
  
  let rows = queryResults.rows;
  while (queryResults.pageToken) {
    queryResults = BigQuery.Jobs.getQueryResults(projectId, jobId, {
      pageToken: queryResults.pageToken
    });
    rows = rows.concat(queryResults.row);
  }
  if (!row) {
    return null;
  }
  
  // convert the result to array
  let data = new Array(rows.length);
  for (let i = 0; i < rows.length; i++) {
    let cols = rows[i].f;
    data[i] = new Array(cols.length);
    for (let j = 0; j < cols.length; j++) {
      data[i][j] = cols[j].v;
    }
  }
  
  return data;
}
```

使い方はこんな感じ：

```javascript
const projectId = ...;
const data = queryTo(`
  #standardSQL
  SELECT 1,2,3 UNION ALL SELECT 4,5,6
`, projectId);
console.log(data); // [[1,2,3],[4,5,6]]
```

クエリ結果が2次元配列で取得できます。フィールド名は捨てられるので、添字で頑張ってアクセスする感じですね。

# 所感

BQのクエリ結果はredashとかtablueとか使って可視化するのが王道ですが、GASを経由してスプレッドシートに転記するのもそれなりに便利だなと思いました。以下その理由。

## Excelに慣れている人にとってスプレッドシートはわかりやすい

特に業務において、社員全員がエンジニアじゃない場合は、モダンなWebアプリの操作感よりも、Officeスイートに似た操作感のほうがウケが良いときがあります。

## GASもBQもGoogleのクラウドサービスなので連携しやすい

そもそも上記GASで使っている `BigQuery` サービスオブジェクトもチェックボックス一つで有効になりますし、アクセスできるBQのプロジェクトはそのスクリプトを実行したユーザーに依存するので、権限のないユーザーが間違えてBQにアクセスすることはありません。

## クエリをJavaScriptで構築できる

個人的にはこれが一番メリットあるかな、と思いました。異なるテーブルから似たようなクエリを作って `join` していい感じの `where` 書いて...のようなSQLを書くことが多いと思います。普通はコピペで対応して結果長大なSQLが出来上がるものですが、GAS(JavaScript)でクエリをかけるので、重複するクエリ部分文字列は変数に入れていい感じに重複を削除できます。

## BQのTIMESTAMP型はJavaScriptのDate互換ではない

1点だけ躓いたのがこれ。一番めんどくさくない変換は string にキャストすること。結果は Date コンストラクタに適合します。

# まとめ

GASでBQのクエリ結果を得る方法を共有しました。例では2次元配列になっているため、 `Range` の `setValues` メソッドに突っ込むだけでスプレッドシートに結果を反映させることができます。もし、BQやredashのクエリ結果をExcelやスプレッドシートにコピペして眺めている作業があれば、GASを使ってトイルを撲滅しましょう。

以上です。
