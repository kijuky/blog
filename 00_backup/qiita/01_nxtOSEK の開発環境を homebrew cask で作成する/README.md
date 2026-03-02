https://qiita.com/kijuky/items/d3f1d700e2db5fa65f24

---

# この記事でできること
Mac にて [nxtOSEK](http://lejos-osek.sourceforge.net/jp/whatislejososek.htm) の開発環境を作成できます。

# 環境構築
前提：[homebrew](https://brew.sh/index_ja) をインストールしていること。

```
brew tap kijuky/lego
brew cask install nxtosek
```

# サンプルを動かしてみる

## ビルド

```
cd /usr/local/Caskroom/nxtosek/v218/samples_c++/cpp/Helloworld
make all
```

## 実機に転送

```
./rxeflash
```

# 参考文献
- [nxtOSEK Installation in Mac OS X Lion](http://lejos-osek.sourceforge.net/installation_Mac_OS_X_Lion.htm)
- [ETロボコン開発環境構築 for Mac](https://qiita.com/tac0x2a/items/b1d82050c660935765ef)
