https://qiita.com/kijuky/items/a84568a30ebbf053e05b

---

ごきげんよう。皆さんは[HSP(Hot Soup Processor)](https://hsp.tv/)というプログラム言語をご存知でしょうか？Windowsで簡単にゲームを作れることが特徴ですが、Android/iOS/JavaScriptなどのプラットフォームでも実行させることができます。macOSでは[machsp](https://github.com/dolphilia/machsp)や[wineを使った方法](https://qiita.com/kotambourine/items/4503718f2690af136892)があります。この記事では、linux版をdockerで動かしてみます。

# 使い方

まずは X Window System をインストールします。

```shell
brew install xquarts
startx
```

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/0b9c742b-e91b-ef11-cb6e-a3ce11a5f280.png) | ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/1b669164-566d-0d8e-4f04-b64e10c7b08b.png) |
|:-:|:-:|
| **XQuartz が立ち上がったのを確認** | **設定から「ネットワーク・クライアントからの接続を許可」をチェック** |

インダイレクト設定を有効化します。

```shell
defaults write org.xquartz.X11 enable_iglx -bool true
```

このあと、**macOSを再起動**します。再起動後、環境変数 `DISPLAY` に何らかの値が設定されているのを確認します。

```shell
env | grep DISPLAY
```

Dockerファイルを用意します ~~（そのうちどこかにホストしようと思います）~~。

:::note info
[追記] GitHub Packages にアップロードしました。下記でpullできます。
https://github.com/kijuky/docker-openhsp-linux/pkgs/container/hsp
```shell
docker pull ghcr.io/kijuky/hsp:3.6
```
:::


```Dockerfile:Dockerfile
FROM ubuntu:22.04

# 必要なソフトをインストール
RUN apt update && apt install -y \
    curl \
    libgtk2.0-dev \
    libglew-dev \
    libsdl1.2-dev \
    libsdl-image1.2-dev \
    libsdl-mixer1.2-dev \
    libsdl-ttf2.0-dev \
    libgles2-mesa-dev \
    libegl1-mesa-dev \
    libsdl2-ttf-dev \
    libsdl2-image-dev \
    libsdl2-mixer-dev \
    && apt clean && rm -rf /var/lib/apt/lists/*

# ソースコードをインストール
RUN curl -L https://github.com/onitama/OpenHSP/archive/refs/tags/v3.6.tar.gz | tar zx -C .
WORKDIR "OpenHSP-3.6"
RUN make

ENV LIBGL_ALWAYS_INDIRECT 1

# /usr/bin にシンボリックリンクを作成
RUN cd /usr/bin; \
    ln -s /OpenHSP-3.6/hsed hsed && \
    ln -s /OpenHSP-3.6/hsp3cl hsp3cl && \
    ln -s /OpenHSP-3.6/hsp3dish hsp3dish && \
    ln -s /OpenHSP-3.6/hsp3gp hsp3gp && \
    ln -s /OpenHSP-3.6/hspcmp hspcmp

# デフォルトはエディタを起動
CMD ["hsed"]
```

ビルドします。

```shell
docker build . -t hsp:3.6
```

まずは例にあるようにコンソールアプリを作ってみましょう。

```shell
% echo 'mes "hello world"' >> test.hsp
% docker run --rm -it -v "$(pwd):/root" -w /root hsp:3.6 hspcmp -d -i -u test.hsp
#HSP script preprocessor ver3.6 / onion software 1997-2021(c)
#HSP code generator ver3.6 / onion software 1997-2021(c)
#use UTF-8 strings.
#Code size (20) String data size (21) param size (0)
#Vars (0) Labels (1) Modules (0) Libs (0) Plugins (0)
#No error detected. (total 170 bytes)
% docker run --rm -it -v "$(pwd):/root" -w /root hsp:3.6 hsp3cl test.ax
hello world
```

いい感じですね :tada: 次にエディタも開いてみましょう。

```shell
xhost +
docker run --rm -it -v "$(pwd):/root" -w /root -e DISPLAY=host.docker.internal:0 --ipc=host hsp:3.6
xhost -
```

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/ffd812a5-bd65-f7e6-77ab-e97d2d9b8951.png) |
|:-:|
| **エディタが開いた様子** |

ゲームも開いてみましょう（エディタの日本語が化けてますね...）

| ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/59468/43656575-06e7-6029-e611-14878625e30f.png) |
|:-:|
| **block3.hspを実行した様子** |

ただ、場合によっては、うまく動かないことがあるようです。dockerのログを見ると、X Window System関連のエラーが出ているようです。

```shell
X Error of failed request:  BadAlloc (insufficient resources for operation)
  Major opcode of failed request:  149 (GLX)
  Minor opcode of failed request:  3 (X_GLXCreateContext)
  Serial number of failed request:  90
  Current serial number in output stream:  91
```

```shell
X Error of failed request:  GLXBadContext
  Major opcode of failed request:  149 (GLX)
  Minor opcode of failed request:  5 (X_GLXMakeCurrent)
  Serial number of failed request:  165
  Current serial number in output stream:  165
```

# まとめ

HSPをmacOS上で動かしてみました。コンソールアプリの場合は問題なく動きます。GUIアプリの場合、X Window Systemのエラーが厄介です。ただ、これも適切に解決すれば、ゲームを起動できるくらいには動かすことができました。皆さんもHSPのプログラミングを楽しんでみてください！

# 参考

https://hsp.tv/make/hsp3linux_pi.html

https://blog.aoirint.com/entry/2020/xquartz_docker/

https://unskilled.site/docker%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%AE%E4%B8%AD%E3%81%A7gui%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E8%B5%B7%E5%8B%95%E3%81%95%E3%81%9B%E3%82%8B/

https://unix.stackexchange.com/a/642418

https://github.com/XQuartz/XQuartz/issues/144

https://web.chaperone.jp/w/index.php?Apple

# 追記

要望はこちら（Dockerfileも随時更新しています）

https://github.com/kijuky/docker-openhsp-linux
