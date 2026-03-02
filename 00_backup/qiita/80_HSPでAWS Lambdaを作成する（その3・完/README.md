https://qiita.com/kijuky/items/dc99c7c155d2ab8be4ec

---

[これ](https://qiita.com/kijuky/items/295273e3127c1857c70f)の続き。

デバッグしたところ、hsp3clで環境変数HOMEを取得する処理があったが、AWS Lambdaには環境変数HOMEが設定されておらず、NULLが返ってきて（おそらくmemcpyで）セグメンテーションフォールトが起こる、という流れだった。

https://x.com/kijuky/status/1932320904366920057

辛かったのが、デバッグ中はセグフォが起こる原因がわからなかったので、オブジェクトファイルを1つずつリンクを外していったり、デバッグプリントをしたりを逐一本番Lambdaにデプロイして確認するのが辛かった。原因がわかれば、rieでも再現可能なんだけどね。。。

というわけで、環境変数HOMEをLAMBDA_TASK_ROOTで設定することで、起動するようになりましたとさ。

https://github.com/kijuky/docker-openhsp-linux/blob/main/amazonlinux2/lambda/bootstrap#L4

エコーサンプルは、以前の記事で共有したリポジトリを参考にしてください。

https://github.com/kijuky/hsp-lambda
