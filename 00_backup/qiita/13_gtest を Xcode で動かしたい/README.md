https://qiita.com/kijuky/items/4a649e90322ac3802aa1

---

# gtest を Xcode で動かしたい

単にリンクを貼るのではなくて、XCUnit の一部として gtest を動かすという、ちょっとマニアックな記事。

## gtest/gtest.h を偽装する。

一般的な gtest は #include <gtest/gtest.h> があるので、ここを接合点として区切る。

区切った後につなぎ合わせるヘッダはこんな感じ

```cpp
#define TEST_F(classname, testname) - (void) test ## testname
namespace testing {
class Test {
}
}
```

## Unit Test クラスを作成する。

Xcode から Unit Test クラス（言語は Objective-C）を作成する。

その後、名前を *.m から *.mm に変更し、Objective-C++ とする。

次に @implementation ~ @end の間に #include "~.cpp" を追加する。

