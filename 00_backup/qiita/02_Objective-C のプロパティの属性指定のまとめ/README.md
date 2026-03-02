https://qiita.com/kijuky/items/86f37aedb67d8c0a27ef

---

# 追記

こっちの記事の方がよっぽどまとまってた

https://qiita.com/uasi/items/80660f9aa20afaf671f3

# はじめに

Objective-C にはプロパティという機能があり、これには属性を設定することができます。この記事ではプロパティに設定する属性についてまとめます。

# まとめ

下記に結論をまとめます。詳細は各章で説明します。

- 常に nonatomic を使う
- assign/retain/strong は使わない
- Delegate は weak を使う
- getter/setter は極力使わない
- nonnull は使わず、 nullable のみ使う
- 不変クラスは copy を使う。可変クラスは copy を使わない
- 読み込みプロパティは、ヘッダで readonly を使い、実装で readwrite を使う
- class を使う場合は自動生成されないことに注意する

## 常に nonatomic を使う

一般に、プロパティへのアクセスは原子性(atomic)を担保するために、ロック処理が自動的に生成されます。プロパティへのアクセスのみがロック処理の対象となるため、あまり有効活用できません。ロックが必要な場合は自前でロック処理を書いたほうが良いです。
ロック処理の自動生成を抑制するために nonatomic を使います。

## assign/retain/strong は使わない

assign/retain は strong/weak に置き換わりました。また、strong は省略可能なので、これら 3 つの属性は指定しません。

## Delegate は weak を使う

Delegate など、所有権をもたないプロパティは weak を使います。

## getter/setter は使わない

よほど特殊な状況でない限り、getter/setter のアクセサ名を変えるべきではありません。

## nonnull は使わず、nullable のみ使う

基本的には全てのオブジェクトは nonnull であるほうが使いやすいです。そのため、コード上では `NS_ASSUME_NONNULL_BEGIN` と `NS_ASSUME_NONNULL_END` で囲み、nonnull を明示せず、nullable なプロパティを明示すると良いです。
最新の Xcode では、デフォルトで `NS_ASSUME_NONNULL_BEGIN` と `NS_ASSUME_NONNULL_END` で囲われるようになるので、このマクロはもはや意識しなくて良くなりました。

## 不変クラスは copy を使う。可変クラスは copy を使わない

ここでいう「不変クラス」は、クラス階層の中で Mutable クラスを持つクラスのことです。例えば NSString や NSArray になります。これらの可変クラスは NSMutableString, NSMutableArray となります。
不変クラスをプロパティとする場合は、必ず copy　属性をつける必要があります。これはサブクラスである可変クラスを設定された場合、そのプロパティがその他の場所で変更されてしまうからです。

```objective-c:Test.m
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic) NSString *foo;
@end

@implementation Hoge
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Hoge *hoge = [Hoge new];
        NSMutableString *foo = [@"foo" mutableCopy];
        hoge.foo = foo;
        NSLog(@"hoge.foo: %@", hoge.foo); // hoge.foo: foo
        [foo setString:@"bar"];
        NSLog(@"hoge.foo: %@", hoge.foo); // hoge.foo: bar
    }
    return 0;
}

NS_ASSUME_NONNULL_END
```

これを防ぐためには foo プロパティに copy 属性をつけます。

```objective-c:Test.m
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic, copy) NSString *foo;
@end

@implementation Hoge
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Hoge *hoge = [Hoge new];
        NSMutableString *foo = [@"foo" mutableCopy];
        hoge.foo = foo; // ここで hoge.foo = [foo copy] となる。
        NSLog(@"hoge.foo: %@", hoge.foo); // hoge.foo: foo
        [foo setString:@"bar"];
        NSLog(@"hoge.foo: %@", hoge.foo); // hoge.foo: foo
    }
    return 0;
}

NS_ASSUME_NONNULL_END
```

一方、可変クラスをプロパティの型として使う場合、copy　属性を使ってはいけません。copy 属性を使うことで副作用のあるメソッドへのアクセスができないインスタンスがプロパティに設定されます。プロパティの型は可変クラスであるため、副作用のあるメソッドへのアクセスはコンパイラでチェックできません。

```objective-c:Test.m
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic, copy) NSMutableString *foo;
@end

@implementation Hoge
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Hoge *hoge = [Hoge new];
        NSMutableString *foo = [@"foo" mutableCopy];
        hoge.foo = foo; // ここで hoge.foo = [foo copy] となる。
        NSLog(@"hoge.foo: %@", hoge.foo);
        [hoge.foo setString:@"bar"]; // signal SIGABORT
        NSLog(@"hoge.foo: %@", hoge.foo);
    }
    return 0;
}

NS_ASSUME_NONNULL_END
```

このケースでは、可変クラス型のプロパティに copy 属性をつけないことで回避できます。

```objective-c:Test.m
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic) NSMutableString *foo;
@end

@implementation Hoge
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Hoge *hoge = [Hoge new];
        NSMutableString *foo = [@"foo" mutableCopy];
        hoge.foo = foo;
        NSLog(@"hoge.foo: %@", hoge.foo);
        [hoge.foo setString:@"bar"];
        NSLog(@"hoge.foo: %@", hoge.foo);
    }
    return 0;
}

NS_ASSUME_NONNULL_END
```

## 読み込みプロパティは、ヘッダで readonly を使い、実装で readwrite を使う

ヘッダに readonly のみで定義してしまうと、自動生成メソッドはゲッターしか生成されません。実装側でセッターがないと不便なことが多いので、無名カテゴリに readwrite で同名のプロパティを宣言します。

```objective_c:Hoge.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Hoge : NSObject
@property(nonatomic, readonly) NSString *fuga;
@end

NS_ASSUME_NONNULL_END
```
```objective_c:Hoge.m
#import "Hoge.h"

NS_ASSUME_NONNULL_BEGIN

@interface Hoge ()
@property(nonatomic, readwrite, copy) NSString *fuga;
@end

@implementation Hoge
@end

NS_ASSUME_NONNULL_END
```

## class を使う場合は自動生成されないことに注意する

Xcode の最新のバージョンでは `class` という属性が使えます。主にシングルトンクラスのプロパティである `sharedInstance` の実装に使われます。
ただし、この属性がついたプロパティは他のプロパティと違って自動生成されないことに注意してください。

```objective-c:Singleton.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Singleton : NSObject
@property(class, nonatomic, readonly) Singleton *sharedSingleton;
@end

NS_ASSUME_NONNULL_END
```
```objective-c:Singleton.m
#import "Singleton.h"

NS_ASSUME_NONNULL_BEGIN

static Singleton *_sharedSingleton;

@implementation Singleton

+ (Singleton *)sharedSingleton {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedSingleton = [Singleton new];
    });
    return _sharedSingleton;
}

@end

NS_ASSUME_NONNULL_END
```

# 最後に

社内で Objective-C のプロパティに指定する属性についての指針がないことが嘆かれていたので、まとめてみました。個人的には、今更 Objective-C のプロパティの属性に悩むのではなく、Swift 移行が進んでそもそも悩む必要がなくなるといいなと思っています。
