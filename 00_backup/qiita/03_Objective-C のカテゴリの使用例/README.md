https://qiita.com/kijuky/items/10ec73ebf9dd423e8084

---

# はじめに

Objective-C には、既存の型にメソッドを追加することができる「カテゴリ」と呼ばれる機能が存在します。この記事ではカテゴリの便利な使用例を紹介します。

## 非公開メソッドを用意する

Effective Objective-C でも紹介されている使い方です。※Java では、同様な機能を package-private なメソッドとして用意します。

```objective_c:Hoge.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
- (NSString *)publicMethod;
@end 

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge+Private.h
#import "Hoge.h"

NS_ASSUME_NONNULL_BEGIN

@interface Hoge (Private)
- (NSString *)privateMethod;
@end 

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge.m
#import "Hoge+Private.h"

NS_ASSUME_NONNULL_BEGIN

@implementation Hoge

- (NSString *)publicMethod {
    return @"publicMethod";
}

@end 

@implementation Hoge (Private)

- (NSString *)privateMethod {
    return @"privateMethod";
}

@end

NS_ASSUME_NONNULL_END
```

利用する場合、`Hoge.h` は Public に、 `Hoge+Private.h` は Project として登録します。ライブラリ内で Private カテゴリにあるメソッドを利用する場合は `Hoge+Private.h` をインポートします。

## Deprecated なメソッドを隔離する

ライブラリが進化していくと、APIを刷新したいなどの理由で、廃止したいメソッドができてきます。この廃止したいメソッド達をカテゴリとして分けることで、メインのAPIだけを1ファイルにまとめることができます。

```objective_c:Hoge.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
- (NSString *)newMethod;
@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge.m
#import "Hoge.h"

NS_ASSUME_NONNULL_BEGIN

@implementation Hoge

- (NSString *)newMethod {
    return @"newMethod";
}

@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge+Deprecated.h
#import "Hoge.h"

NS_ASSUME_NONNULL_BEGIN

@interface Hoge (Deprecated)
- (NSString *)oldMethod DEPRECATED_MSG_ATTRIBUTE("use newMethod instead.");
@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge+Deprecated.m
#import "Hoge+Deprecated.h"

NS_ASSUME_NONNULL_BEGIN

@implementation Hoge (Deprecated)

- (NSString *)oldMethod {
    return @"oldMethod";
}

@end

NS_ASSUME_NONNULL_END
```

Deprecated な実装が分かれるので、`DEPRECATED_MSG_ATTRIBUTE` などを使ってリッチな警告を出すようにしても、メインの処理がごちゃごちゃしません。実際に廃止する際も、このファイルを削除するだけで良いです。

似たようなものとして、デバッグ時のみに使う Debug カテゴリや、テスト時のみに使う Test カテゴリ、互換性を維持する Compat カテゴリなども有効です。

## C++ 実装をラップする

C++ ライブラリを利用する場合、Swift プロジェクトであっても Objective-C のラッパー層が必要になります。この時、C++ に関する実装をカテゴリにまとめることで、Swift プロジェクトでも利用できる Objective-C クラスを実装できます。

```objective_c:Hoge.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic, readonly) NSString *text;
- (instancetype)init NS_UNAVAILABLE;
@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge.m
#import "Hoge.h"

NS_ASSUME_NONNULL_BEGIN

@implementation Hoge
@end

NS_ASSUME_NONNULL_END
```
```cpp:CHoge.hpp
#include <string>

class CHoge {
  public:
    std::string text;
};
```
```objective_c:Hoge+Cpp.hpp
#import "Hoge.h"

#import "CHoge.hpp"

NS_ASSUME_NONNULL_BEGIN

@interface Hoge (Cpp)
- (instancetype)initWithCHoge(const CHoge &)cHoge;
@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge+Cpp.mm
#import "Hoge+Cpp.hpp"

NS_ASSUME_NONNULL_BEGIN

@implementation Hoge (Cpp)

- (instancetype)initWithCHoge(const CHoge &)cHoge {
    self = [super init];
    if (self) {
        _text = @(cHoge.text.c_str());
    }
    return self;
}

@end

NS_ASSUME_NONNULL_END
```

`Hoge.h` を使うだけであれば C++ が出てこないので、このクラスは Swift で利用できます。

# 最後に

他にも有用な使い方があれば追記していきたいと思います。
