https://qiita.com/kijuky/items/295273e3127c1857c70f

---

OpenHSPでAWS Lambdaを作る記事です。[以前の記事](https://qiita.com/kijuky/items/fffac99c93fef2382866)から進展したので、その報告です。

# 0. Lambdaでカスタムランタイムを作るには

https://goropikari.hatenablog.com/entry/aws_lambda_custom_runtime

上の記事に書いてありますが、AWSが提供していない言語をLambdaで使いたい場合、bootstrapを用意する必要があります。

# 1. bootstrap を用意する。

```diff_shell:bootstrap
#!/bin/sh
set -e

: "${HSP_SOURCE_NAME:=sample}"
HSP_OBJECT_FILE=${HSP_SOURCE_NAME}.ax

RUNTIME_URL="http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01"

post() {
    request_id="$1"
    response="$2"
    url="${RUNTIME_URL}/runtime/invocation/${request_id}/response"
    curl -X POST "${url}" -d "${response}"
}

while true
do
    HEADERS="$(mktemp)"
    EVENT_DATA=$(curl -sS -LD "${HEADERS}" -X GET "${RUNTIME_URL}/runtime/invocation/next")
+   RESPONSE=$(echo "${EVENT_DATA}" | hsp3cl "${HSP_OBJECT_FILE}" | tr -d '\r' | sed '1d')
    REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "${HEADERS}" | tr -d '[:space:]' | cut -d: -f2)
    post "${REQUEST_ID}" "${RESPONSE}"
done
```

コードはほとんどサンプルと同じです。肝はここで

```sh
    RESPONSE=$(echo "${EVENT_DATA}" | hsp3cl "${HSP_OBJECT_FILE}" | tr -d '\r' | sed '1d') # 1行目のエラー(gpiod initalize failed.)を消す
```

- `echo "${EVENT_DATA}"` - ランタイムの標準入力にラムダの引数を設定
- `hsp3cl "${HSP_OBJECT_FILE}"` - HSPの実行
- `tr -d '\r'` - HSPは改行を`\r\n`で出力するのですが、Linux(Ubuntu)で動くcurlはこの改行を認識できないようです。`\r`を消すことでLinuxの改行に適合します。
- `sed '1d'` - HSP3.7betaから[GPIO](https://ja.wikipedia.org/wiki/GPIO)の初期化が実行されるようになり、起動すると1行目に`gpiod initalize failed.`と出力されます。LambdaでGPIOは使わないので出力を消します。

# 2. Dockerfileの修正

```Dockerfile:Dockerfile
FROM --platform=linux/amd64 ghcr.io/kijuky/hsp:3.7beta10

ENV LAMBDA_RUNTIME_DIR=/var/runtime \
    LAMBDA_TASK_ROOT=/var/task

# HSPソースコードのビルド
WORKDIR ${LAMBDA_TASK_ROOT}
ARG HSP_SOURCE_NAME=sample
ENV HSP_SOURCE_NAME=${HSP_SOURCE_NAME}
COPY ${HSP_SOURCE_NAME}.hsp ./
RUN hspcmp --compath=/OpenHSP-3.7beta10/common/ -d -i -u ${HSP_SOURCE_NAME}.hsp

# bootstrapの起動
COPY bootstrap ${LAMBDA_RUNTIME_DIR}/bootstrap
RUN chmod +x ${LAMBDA_RUNTIME_DIR}/bootstrap
ENTRYPOINT ["${LAMBDA_RUNTIME_DIR}/bootstrap"]
```

[前回](https://qiita.com/kijuky/items/fffac99c93fef2382866#2-docker%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8%E3%81%AE%E4%BD%9C%E6%88%90)はHSPスクリプトを起動していましたが、今回はbootstrapを起動します。

# 3. 実行する

テスト実行してみます。

```
START RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx Version: $LATEST
Segmentation fault (core dumped)
response: 
```

[aws-lambda-rie](https://aws.amazon.com/jp/builders-flash/202104/new-lambda-container-development-2/)だと実行できましたが、実際のlambdaで実行すると`Segmentation fault (core dumped)`が出てしまいました。残念。。（やっぱりGPIOまわりがまずいのかなぁ？）

# 補足

テスト用のリポジトリを用意しておきます。興味が出た方はフォークして調査研究してみてください...

https://github.com/kijuky/hsp-lambda
