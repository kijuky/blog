https://qiita.com/kijuky/items/fffac99c93fef2382866

---

[OpenHSPのdockerイメージを作った](https://qiita.com/kijuky/items/a84568a30ebbf053e05b)のですが、何か有効活用できないかと思い、これをベースイメージとしてAWS Lambdaを構築してみました。

結論としては、構築できませんでした。作業ログを書いておきます。

# 0. Lambda実行用ユーザーを作成

AWS CLIをインストールします。

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

いい感じに設定します。

https://docs.aws.amazon.com/ja_jp/cli/v1/userguide/cli-chap-configure.html

テスト用にLambdaを起動するIAMユーザーを作成します。

```bash:console
aws iam create-role \
  --role-name lambda-execute-hsp \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "lambda.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
aws iam attach-role-policy \
  --role-name lambda-execute-hsp \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy \
  --role-name lambda-execute-hsp \ 
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

# 1. コンソール用ソースコードの作成

こんな感じのコードを用意します。コメントで `json` と言っていますが、特にJSONを解析してないです。

```hsp:sample.hsp
; sample.hsp - stdinからJSON文字列を読み込んで表示する（hspinet不要）

#include "hsp3cl.as"

; 読み込み用のバッファと一時変数
sdim json_str, 4096, 1
sdim line_str, 1024

repeat
	; 1行ずつ読み込む
	input line_str, 1024, 1
	if strsize = 1 : break ; EOFで終了
	json_str += line_str + "\n"
loop

; 読み込んだJSON文字列をそのまま表示
mes json_str
```

# 2. Dockerイメージの作成

こんなDockerfileを用意します。

```Dockerfile:Dockerfile
FROM ghcr.io/kijuky/hsp:3.7beta10@sha256:8449c42b11c48ac3caa04130911a6ed5411774d4b3e99b2ed8f48297948555d9 # amd64イメージ

COPY sample.hsp ./
RUN hspcmp --compath=/OpenHSP-3.7beta10/common/ -d -i -u sample.hsp

CMD ["hsp3cl", "sample.ax"]
```

ビルドします。Lambdaで使う都合上、Image Indexを作らないようにします。`DOCKER_BUILDKIT=0` で buildx を使わないようにします。

```bash:console
DOCKER_BUILDKIT=0 DOCKER_DEFAULT_PLATFORM=linux/amd64 docker build -t echo-lambda-hsp:latest .
```

# 3. ECRにイメージをプッシュ

プライベートリポジトリにイメージをプッシュします。Lambdaでは公開イメージを利用できません。

```bash:console
aws ecr create-repository \
  --repository-name echo-lambda-hsp
```

出力に作ったリポジトリがレポートされます。ここでは `000000000000.dkr.ecr.us-east-1.amazonaws.com/echo-lambda-hsp` とします作ったリポジトリにプッシュします。

```bash:console
docker tag echo-lambda-hsp:latest 000000000000.dkr.ecr.us-east-1.amazonaws.com/echo-lambda-hsp:latest
docker push 000000000000.dkr.ecr.us-east-1.amazonaws.com/echo-lambda-hsp:latest
```

# 4. Lambdaを作成

```bash:conosle
aws lambda create-function \
  --function-name echo-lambda-hsp \
  --package-type Image \
  --code ImageUri=000000000000.dkr.ecr.us-east-1.amazonaws.com/echo-lambda-hsp:latest \
  --role arn:aws:iam::000000000000:role/lambda-execute-hsp
```

# 5. テスト実行

```bash:console
aws lambda invoke \
  --function-name echo-lambda-hsp \
  --payload '{"key": "value"}' \                                           
  --cli-binary-format raw-in-base64-out \
  response.json && cat response.json
```

これでエコーしてくれると良かったんですが、エラーが返ってきてました。

```json:console
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
{"errorType":"Runtime.ExitError","errorMessage":"RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx Error: Runtime exited with error: signal: segmentation fault"}
```

```:cloudwatch
INIT_REPORT Init Duration: 154.03 ms	Phase: init	Status: error	Error Type: Runtime.ExitError
INIT_REPORT Init Duration: 99.65 ms	Phase: invoke	Status: error	Error Type: Runtime.ExitError
START RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx Version: $LATEST
RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx Error: Runtime exited with error: signal: segmentation fault
Runtime.ExitError
END RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
REPORT RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx	Duration: 140.05 ms	Billed Duration: 141 ms	Memory Size: 128 MB	Max Memory Used: 4 MB	
```

`INIT_REPORT` はそもそもコード実行まで辿り着いていないようです。

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/lambda-runtime-environment.html

`Phase: init Status: error` でググると、ECRのポリシー不足という記事も見つけたのですが、うまくいかなかったです。

https://rkd3.dev/post/lambdaecrperm/

# まとめ

LinuxベースのHSPがDockerイメージで利用できるようになったため、AWS Lambda上で実行してみようと試みました。しかしながら、初期化のタイミングで失敗してしまいました。私は失敗してしまいましたが、HSPやAWS Lambdaに興味がある人はぜひトライしてみてください！
