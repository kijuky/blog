https://qiita.com/kijuky/items/d9cd953be9c32c8e6378

---

# はじめに

最近、**型レベルプログラミング（Type-level Programming）** という技術にどっぷりハマりました。普段のWeb開発ではあまり使うことのないこの技術ですが、使ってみるととても強力で、しかも面白い！

この記事では「型で書くDSLのコンパイラ」として Scala 3 の型レベル機能を使って、物理の次元解析を題材に遊んでみた記録を残します。

# 型レベルプログラミングとは？

通常のプログラミングは「値」に対してロジックを書きます。しかし、**型レベルプログラミングは「型」に対してロジックを書く**ものです。型そのものをデータとして扱い、型を操作することで**より強い静的保証や構文レベルの制約**をプログラムに埋め込むことができます。

Scala 3 では特にこれがやりやすくなりました。できることの例：

- 型による DSL（ドメイン特化言語）の文法制約
- 型の再帰的変換（e.g. 正規化、変換、型パスの構築）
- コンパイル時に `a + b` のような演算の式木を型で保持・操作
- 型に条件分岐や合成を定義できる
- そして**コンパイル時エラーによって誤用を防げる！**

つまり、「型＝構文」「型レベル関数＝構文解析器」と見なせば、DSLのコンパイラが型で書けるのです。

# 型レベルプログラミングの主要3ツール

https://www.scala-lang.org/2021/02/26/tuples-bring-generic-programming-to-scala-3.html

## 1. HList として進化した Tuple

Scala 3 の Tuple は、かつて shapeless や Miles Sabin の HList によって実現されていた**異種長リスト（Heterogeneous List）** の機能を、標準の型としてサポートします。

```scala
val t0: Tuple = EmptyTuple    // ()
val t1: Tuple = Tuple1(1)     // (Int) ※要素が1つの場合は `(1)` とは書けない
val t2: Tuple = (1, 2.0)      // (Int, Double)
val t3: Tuple = (1, 2.0, "3") // (Int, Double, String)
```

- `Tuple.Append` や `Tuple.Zip` など、標準でタプル操作が提供されている
- 型として `Tuple1[A]`, `(A, B, C)` などが単なる構文糖衣であり、型操作ができる
- `EmptyTuple` をベースに、型レベルのリストとして再帰展開できる

この型タプルを再帰的に処理することで、型リストの変換、マッピング、合成などが自由自在に行えます。

## 2. 再帰的な match types による型関数定義

Scala 3 では型に対する `match` が導入され、これを用いることで 型に対する関数定義（型関数） が書けます。これによって型の変換・正規化・比較が型レベルで可能になります。

例として、Tuple に含まれる Int 型の個数を数えてみます。

```scala
import scala.compiletime.ops.int.+
type CountInt[T <: Tuple] <: Int = T match
  case EmptyTuple  => 0                  // 終了条件。ここで `0` は値ではなく型
  case Int *: tail => 1 + CountInt[tail] // Int型なら結果をインクリメント
  case _ *: tail   => CountInt[tail]     // Int型以外なら結果をそのまま返す
```

- `E match` によって型構造をパターンマッチ
- 結果として `Tuple` 型（次元ベクトルなど）を構築
- 再帰的に型構造を辿ることが可能

このように match types によって「型で書かれた式」を解析し、「構造を持った型」に変換できます。

## 3. `=:=` による型の一致証明と暗黙解決

型レベルの比較を行うための基本ツールが `=:=` です。これは `A =:= B` のような型で表され、「AとBが等しいならば使用可能」とする制約になります。

例えば、先ほどの例を使い、Int型が含まれていないTupleのみを受け付ける関数を書いてみます。

```scala
def notIncludedInt[A <: Tuple](a: A)(using ev: CountInt[A] =:= 0): Unit = ???
```

ev は evidence の略らしいです。

```scala
notIncludedInt(EmptyTuple) // OK
notIncludedInt(Tuple(1.0)) // OK
notIncludedInt((1.0, "2")) // OK

notIncludedInt(Tuple(1)) // Cannot prove that CountInt[Int *: EmptyTuple] =:= (0 : Int).
notIncludedInt((1, 2.0)) // Cannot prove that CountInt[(Int, Double)] =:= (0 : Int).
```

- `ev: A =:= B` は「型`A`と型`B`が同じである」という型レベルの等価性の証明
- 暗黙引数として与えることで、コンパイル時に一致するかどうかを判定可能
- `summon[A =:= B]` で手動証明を要求することもできる

これによって、**「引数の型がevを満たしているか？」をコンパイル時に検査できる**ようになります。そのままではチェックできない型を、match typesによってScalaの型システムで比較可能な型に変換して、`using ev: =:=[A, B]`でチェックする、というのが基本戦略になります。

# 実装例：物理の次元解析を型でやってみた

もう少し実践的な例を提示して、型レベルプログラミングがどのようにDSL構築に役立つかをみていきます。ここでは物理の次元解析をしてみます。

[物理量](https://ja.wikipedia.org/wiki/%E9%87%8F%E3%81%AE%E6%AC%A1%E5%85%83)は「長さ」「時間」「質量」などの [**次元（Dimension）**](https://ja.wikipedia.org/wiki/%E9%87%8F%E3%81%AE%E6%AC%A1%E5%85%83) から構成されます。

```scala:Dim.scala
/** 次元 */
enum Dim {
  /** 長さ */
  case Length
  /** 時間 */
  case Time
  /** 質量 */
  case Mass
}
```

そして「速度」は `Length / Time`、粘性係数は `Mass / (Length * Time)` というように式の形で次元が決まります。次元を式の形で表してみます。

```scala:DimExp.scala
/** 次元式 */
enum DimExp {
  /** 基本次元 */
  case Base[D <: Dim]()
  /** 乗法 */
  case Mul[A <: DimExp, B <: DimExp]()    // A * B
  /** 除法 */
  case Div[N <: DimExp, D <: DimExp]()    // Numerator / Denominator
  /** 指数 */
  case Pow[B <: DimExp, E <: Double]() // Base ^ Exponent
}
```

これを型で書くとこうなります：

```scala
// 基本次元
type Length = DimExp.Base[Dim.Length.type]
type Time = DimExp.Base[Dim.Time.type]
type Mass = DimExp.Base[Dim.Mass.type]

// 組み立て次元
type Density = DimExp.Div[Mass, DimExp.Pow[Length, 3.0]]
type Velocity = DimExp.Div[Length, Time]
type Viscosity = DimExp.Div[Mass, DimExp.Mul[Length, Time]]
```

物理の量を表すために数値と次元を持つ型 Quantity を定義します。

```scala:Quantity.scala
case class Quantity[D <: DimExp](value: Double) {
  def *[D2 <: DimExp](that: Quantity[D2]): Quantity[DimExp.Mul[D, D2]] =
    Quantity(this.value * that.value)

  def /[D2 <: DimExp](that: Quantity[D2]): Quantity[DimExp.Div[D, D2]] =
    Quantity(this.value / that.value)
}
```

これで次元付きの計算ができるようになります。


```scala
val rho: Quantity[Density] = Quantity(1000.0)
val v: Quantity[Velocity] = Quantity(2.0)
val L: Quantity[Length] = Quantity(0.5)
val mu: Quantity[Viscosity] = Quantity(1.0)

val re = rho * v * L / mu // Quantity[DimExp.Div[DimExp.Mul[DimExp.Mul[Density, Velocity], Length], Viscosity]]
```

実際の[レイノルズ数](https://ja.wikipedia.org/wiki/%E3%83%AC%E3%82%A4%E3%83%8E%E3%83%AB%E3%82%BA%E6%95%B0)は[無次元量](https://ja.wikipedia.org/wiki/%E7%84%A1%E6%AC%A1%E5%85%83%E9%87%8F)なので、この次元が無次元であることを確かめてみましょう。分子に出てきた`Dim`の数を数え、分母に出てきた`Dim`の数を引いて0になることを確かめます。`Dim`を数え上げるための一時的な型を定義します。

```scala
case class DimPower[D <: Dim, E <: Double]() // 次元の数を数える
type DimMap = Tuple // 式の次元を表す
```

再帰 match types を使って `DimExp` を演算します。

```scala
type ToDimMap[D <: DimExp] <: DimMap = D match
  case DimExp.Base[d] =>
    // 基本次元は (Dim, 1.0) とする。
    Tuple1[DimPower[d, 1.0]]
  case DimExp.Mul[a, b] =>
    // 二つの次元式の掛け算の場合、それぞれの次元マップを足し合わせる
    MergeDimMap[ToDimMap[a], ToDimMap[b]]
  case DimExp.Div[n, d] =>
    // 二つの次元式の割り算の場合、分母の部分は全ての指数を反転させる
    MergeDimMap[ToDimMap[n], NegateDimMap[ToDimMap[d]]]
  case DimExp.Pow[b, e] =>
    // 次元式の指数の場合、指数部分をe倍する
    ScaleDimMap[ToDimMap[b], e]
```

簡単なものからやっていきましょう。`NegateDimMap`と`ScaleDimMap`はそれぞれ指数部分の掛け算になります。

```scala
import scala.compiletime.ops.double.*
type NegateDimMap[M <: DimMap] <: DimMap = M match
  case EmptyTuple => EmptyTuple
  case DimPower[d, n] *: tail => 
    DimPower[d, Negate[n]] *: NegateDimMap[tail]
type ScaleDimMap[M <: DimMap, N <: Double] <: DimMap = M match
  case EmptyTuple => EmptyTuple
  case DimPower[d, n] *: tail =>
    DimPower[d, n * N] *: ScaleDimMap[tail, N]
```

`MergeDimMap`は再帰部分と演算部分に分けます。

```scala
import scala.compiletime.ops.double.*
// 再帰部分
type MergeDimMap[A <: DimMap, B <: DimMap] <: DimMap = A match
  case EmptyTuple => B
  case DimPower[d, n] *: atail =>
    MergeOneDimMap[d, DimPower[d, n], MergeT[atail, B]]
// 演算部分
type MergeOneDimMap[D <: Dim, P <: DimPower[D, ?], M <: DimMap] <: DimMap =
  (P, M) match
    case (DimPower[d, n], EmptyTuple) =>
      // 同じ次元が見つからなかったので、新規エントリの追加
      Tuple1[DimPower[d, n]]
    case (DimPower[D, n], DimPower[D, m] *: tail) =>
      // 同じ次元なら指数を足す
      DimPower[D, n + m] *: tail
    case (DimPower[d, n], head *: tail) =>
      // 同じ次元が見つかるまで再帰
      head *: MergeOneDimMap[d, DimPower[d, n], tail]
```

これで式に現れる次元を `DimMap`(`Tuple`) にできました。最後に `Dim` を渡して比較可能な型を生成します。ここでは、型に現れる次元の順序を無視するため、交差型を使います。

```scala
type Dimensionless = Any
type EvalDimension[D <: DimExp] = ToIntersection[ToDimMap[D]]
type ToIntersection[T <: DimMap] <: Any = T match
  case EmptyTuple => Dimensionless
  case DimPower[_, 0.0] *: tail => ToIntersection[tail]
  case DimPower[d, n] *: t => (d, n) & ToIntersection[t]
```

少し動かしてみましょう。定義済みの組み立て次元とかはいい感じに比較できてそうです。

```scala
summon[EvalDimension[Length] =:= (Dim.Length.type, 1.0)]
summon[EvalDimension[Velocity] =:= ((Dim.Length.type, 1.0) & (Dim.Time.type, -1.0))] // カッコが必要（外側のカッコはタプルのカッコではない）
summon[EvalDimension[Velocity] =:= ((Dim.Time.type, -1.0) & (Dim.Length.type, 1.0))] // 順序は関係ない
summon[EvalDimension[Density] =:= ((Dim.Mass.type, 1.0) & (Dim.Length.type, -3.0))]
summon[EvalDimension[Viscosity] =:= ((Dim.Mass.type, 1.0) & (Dim.Length.type, -1.0) & (Dim.Time.type, -1.0))]
```

無次元量であるレイノルズ数はこんな感じになります。

```scala
val re = ... // Quantity[D]

// QuantityからDimを取り出して結果の型を得る
type EvalQuantityDimension[Q <: Quantity[?]] <: Any = Q match
  case Quantity[d] => EvalDimension[d]

type Re = re.type
summon[EvalQuantityDimension[Re] =:= Dimensionless]
```

# まとめ

Scala 3 の型レベルプログラミングは「型でDSLのコンパイラを書く」感覚に近いです。

- 計算式を型として保持することで、安全に次元演算を行える
- 無次元量の区別、演算の順序や構文木の差異までコンパイル時に追跡できる
- 表示や比較、変換まで型と given を活用して安全に実装できる

Web開発ではなかなか触れる機会のない型レベル世界ですが、**興味がある人にはぜひ試してほしい世界です**。DSLや数式処理、あるいは型安全な構造を考える人には最高の遊び場になるはず！

# おわりに

今回使った実装は OSS として [GitHub: kijuky/qantica](https://github.com/kijuky/qantica) に公開しています。
興味がある方はぜひスター・コントリビュートお待ちしています！
