# fp-in-scala-4

例外を使わないエラー処理

---

## 4.1 例外の光と影

---

### 例1

```
def failingFn(i: Int): Int = {
  val y: Int = throw new Exception("fail!")
  try {
    val x = 42 + 5
    x + y
  } catch { 
    case e: Exception => 43 
  }
}
```

---

### 結果

```
scala> failingFn(12)
java.lang.Exception: fail!
  at .failingFn(<console>:11)
  ... 33 elided

```

---

### 例2

```
def failingFn(i: Int): Int = {
  try {
    val x = 42 + 5
    x + ((throw new Exception("fail!")): Int)
  } catch {
    case e: Exception => 43
  }
}
```

---

### 結果

```
scala> failingFn(12)
res0: Int = 43
```

### 結論

yが参照透過ではない!

---

### 参照透過な式

コンテキストに依存せず、ローカルでの推論が可能である。

例: 1 + 2

---

### 参照透過ではない式

コンテキストに依存し、よりグローバルな推論が必要となる。

例: throw new Exception("fail")

---

### 2つ問題

1/2 例外ベースの複雑なコードが記述可能

昔の忠告: 例外はエラー処理のみ使用するべきであり、制御フローには使用するべきではない

---

### 2つ問題

2/2 例外が型安全ではない

failingFn, Int => Intの型から例外が発生することなど何もわからないので実行時まで検出できない

---

### 利点もある

エラー処理ロジックの一本化が可能

```
class AbstractServlet {
  ...
  public doPost(Request req, Response res) {
    ...
    catch (LoginException: e) {...}
    catch (ValidateException: e) {...}
    catch (DBException: e) {...}
    catch (IOException: e) {...}
    ...
    catch (Exception: e) {...}
  }
  ...
}

```

---

## 4.2 例外に代わる手法

---

リストの平均を計算する関数

```
def mean(xs: Seq[Double]): Double =
  if (xs.isEmpty)
    throw new ArithmeticException("mean of empty list!")
  else xs.sum / xs.length

```

---

### 一つ目の方法

Double型の偽の値を返すこと

```
def mean(xs: Seq[Double]): Double =
  if (xs.isEmpty) Double.NaN
  else xs.sum / xs.length
```

```
  // java.lang.Double
  /**
   * A constant holding a Not-a-Number (NaN) value of type
   * {@code double}. It is equivalent to the value returned by
   * {@code Double.longBitsToDouble(0x7ff8000000000000L)}.
   */
  public static final double NaN = 0.0d / 0.0;
```

---

### 却下理由

1. エラーが隠れてしまう

2/3/4. (略)

---

### 2つ目の方法

呼び出す元で例外時の戻り値を用意すること

```
def mean(xs: Seq[Double], onEmpty: Double): Double =
  if (xs.isEmpty) onEmpty
  else xs.sum / xs.length
```

---

### 却下理由

呼び出し元処理で分岐できない
※ 一つ目の方法と似ている

---

## 4.3 Option データ型

---

```
sealed trait Option[+A]
case class Some[+A](get: A) extends Option[A]
case object None extends Option[Nothing]
```

Optionを使った実装

```
def mean(xs: Seq[Double]): Option[Double] =
  if (xs.isEmpty) None
  elase Some(xs.sum / xs.length)
```

完全な関数になる!

---

### 4.3.1 Option の使用パターン

- 指定されたキーによるマップ検索
- リストなどで定義されている headOption と lastOption
- ...

---

### Option の基本関数

```
trait Option[+A] {
  def map[B](f: A => B): Option[B])
  def flatMap[B](f: A => Option[B])
  def getOrElse[B >: A](default: => B): B
  def orElse[B >: A](ob: => Option[B])]: Option[B]
  def filter(f: A => Boolean): Option[A]
}
```

---

### EXERCISE 

基本関数を全て実装せよ

---

### ANSWER 4.1

```
  def map[B](f: A => B): Option[B] = this match {
    case Some(a) => Some(f(a))
    case None => None
  }
  def getOrElse[B>:A](default: => B): B = this match {
    case Some(a) => a
    case None => default
  }
  def flatMap[B](f: A => Option[B]): Option[B] = this match {
    case Some(a) => f(a)
    case None => None
  }
  def orElse[B>:A](ob: => Option[B]): Option[B] = this match {
    case Some(_) => this
    case None => ob
  }
  def filter(f: A => Boolean): Option[A] = this match {
    case Some(a) if f(a) => this
    case _ => None
  }

```

### Option の基本関数を使用するシナリオ

Employeeの例

(略)

---

### EXERCISE 4.2

flatMap をベースとして variance 関数を実装せよ

シーケンスの平均をm、シーケンスの各要素をxとすれば、分散は math.pow(x - m, 2) の平均となる
```
  def variance(xs: Seq[Double]): Option[Double]
```

---

### ANSWER 4.2

```
  def variance(xs: Seq[Double]): Option[Double] =
    mean(xs).flatMap(avg => mean(xs.map(x => math.pow(x - avg, 2))))
```

最初の失敗(Empty)が検出される時点で計算は中止される

---

### 4.3.2 Option の合成、リフト、例外指向の APIのラッピング

呼び出し元の Some / None 分岐処理は急がなくでも良い

```
  def lift[A, B](f: A => B): Option[A] => Option[B] = _ map f
```

f を lift した関数

```
  val abs0: Option[Double] => Option[Double] = lift(math.abs)
```

lift を通して math.abs を Option コンテキスト内でもど動作するようになる

---

### 自動車保険料計算の例

```
  def parseInsuranceRateQuote(
    age: String,
    numberOfSpeedingTickets: String
  ): Option[Double] = {
    val optAge: Option[Int] = Try(age.toInt)
    val optTicket: Option[Int] = Try(numberOfSpeedingTickets.toInt)
    insuranceRateQuote(optAge, optTickets)
  }
  
  def Try[A](a: => A): Option[A] =
    try Some(a)
    catch { case e: Exception => None }
```

欠点: fail first できない

---

### EXERCISE 4.3

2項関数を使ってOption型の2の値を結合する総称関数 map2 を記述せよ。

```
  def map2[A,B,C](a: Option[A], b: Option[B])(f: (A, B) => C): Option[C]
```

---

### ANSWER 4.3

```
  def map2[A,B,C](a: Option[A], b: Option[B])(f: (A, B) => C): Option[C] = (a, b) match {
    case (Some(aa), Some(bb)) => Some(f(aa, bb))
    case _ => None
  }
```

```
  def map2[A,B,C](a: Option[A], b: Option[B])(f: (A, B) => C): Option[C] =
    a flatMap (aa => b map (bb => f(aa, bb)))
```

---

### EXERCISE 4.4

Option のリストを 1つのOptionにまとめる sequence 関数を記述せよ。

```
  def sequence[A](a: List[Option[A]]): Option[List[A]]
```

---

### ANSWER 4.4

```
  def sequence[A](a: List[Option[A]]): Option[List[A]] = a match {
    case Nil => Some(Nil)
    case h::t => h.flatMap(hh => sequence(t).map(tt => hh::tt))
  }
```

---

### 残念な例

```
  def parseInts(a: List[String]): Option[List[Int] =
    sequence(a map (i => Try(i.toInt)))
```

リストを2回走査するため効率よくない

---

### EXERCISE 4.5

リストを1回だけの走査で、traverse 関数を実装せよ

```
  def traverse[A, B](a: List[A)(f: A => Option[B]): Option[List[B]]
```

---

### ANSWER 4.5

```
  def traverse[A, B](a: List[A])(f: A => Option[B]): Option[List[B]] = a match {
    case Nil => Some(Nil)
    case h::t => map2(f(h), traverse(t)(f))(_ :: _) // Option[B], Option[List[B]]
  }
```

---

## 4.4 Either データ型

---

### 本章の目的

エラーや例外を通常の値で表し、エラー処理とリカバリに共通するパターンを関数として抽出できるようにすること。

Optionだと表現力が足りない

---

### Eitherの定義

```
  sealed trait Either[+E, +A]
  case class Left[+E](value: E) extends Either[E, Nothing]
  case class Right[+A](value: A) extends Either[Nothing, A]
```

- Right -> 成功 ※ 英語の意味は「正しい」
- Left -> エラー

---

### mean の例


```
  def mean(xs: IndexedSeq[Double]): Either[String, Double] =
    if (xs.isEmpty)
      Left("mean of empty list!")
    else
      Right(xs.sum / xs.length)
```

String -> Exception に変更すると情報力がさらに増える

---

### Try 関数

```
  def Try[A](a: => A): Either[Exception, A] =
    try Right(a)
    case { case e: Exception => Left(e) }
```

---

###  EXERCISE 4.6

Right 値を操作する map, flatMap, orElse, map2 をEither に追加せよ。

```
sealed trait Either[+E, +A] {
  def map[B](f: A => B): Either[E, B] = ???
  def flatMap[EE >: E, B](f: A => Either[EE, B]): Either[EE, B] = ???
  def orElse[EE >: E, B >: A](b: => Either[EE, B]): Either[EE, B] = ???
  def map2[EE >: E, B, C](b: Either[EE, B])(f: (A, B) => C): Either[EE, C] = ???
}
```

---

### ANSWER 4.6

```
sealed trait Either[+E, +A] {
  def map[B](f: A => B): Either[E, B] = this match {
    case Left(e) => Left(e)
    case Right(a) => Right(f(a))
  }

  def flatMap[EE >: E, B](f: A => Either[EE, B]): Either[EE, B] =
    this match {
      case Left(e) => Left(e)
      case Right(a) => f(a)
    }

  def orElse[EE >: E, B >: A](b: => Either[EE, B]): Either[EE, B] =
    this match {
      case Left(_) => b
      case Right(_) => this
    }
  ...
```

---

### ANSWER 4.6-2

```
  ...
  def map2[EE >: E, B, C](b: Either[EE, B])(f: (A, B) => C): Either[EE, C] =
    b.flatMap(bb => this.map(aa => f(aa, bb)))

  def map2_2[EE >: E, B, C](b: Either[EE, B])(f: (A, B) => C): Either[EE, C] =
    for {
      aa <- this
      bb <- b
    } yield f(aa, bb)

  def map2_3[EE >: E, B, C](b: Either[EE, B])(f: (A, B) => C): Either[EE, C] =
    (this, b) match {
      case (Right(aa), Right(bb)) => Right(f(aa, bb))
      case (Left(e), _) => Left(e)
      case (_, Left(e)) => Left(e)
    }
```

---

### EXERCISE 4.7

sequence と traverse を実装せよ。エラーが発生した場合は、最初に検出されたエラーを返す。

```
  def sequence[E,A](es: List[Either[E,A]]): Either[E,List[A]] = ???
  
  def traverse[E,A,B](es: List[A])(f: A => Either[E, B]): Either[E, List[B]] = ???
```

---

### ANSWER 4.7

```
  def sequence[E,A](es: List[Either[E,A]]): Either[E,List[A]] =
    es match {
      case Nil => Right(Nil)
      case h::t => h.flatMap(hh => sequence(t).map(tt => hh::tt))
    }

  def traverse[E,A,B](es: List[A])(f: A => Either[E, B]): Either[E, List[B]] =
    es match {
      case Nil => Right(Nil)
      case h::t => f(h).map2(traverse(t)(f))(_ :: _) // Right(B), Right[List[B]]
    }
```

---

### Either を使ったデータ検証(Validate)

```
object Person {
  def mkName(name: String): Either[String, Name] =
    if (name == "" || name == null) Left("Name is empty.")
    else Right(Name(name))

  def mkAge(age: Int): Either[String, Age] =
    if (age < 0) Left("Age is out of range.")
    else Right(Age(age))

  def mkPerson(name: String, age: Int): Either[String, Person] =
    mkName(name).map2(mkAge(age))(Person(_, _))
}
```

すべてのエラーを拾うにはどうすれば良い?

---

### OPEN ANSWER

```
  There are a number of variations on `Option` and `Either`. If we want to accumulate multiple errors, a simple
  approach is a new data type that lets us keep a list of errors in the data constructor that represents failures:
  
  trait Partial[+A,+B]
  case class Errors[+A](get: Seq[A]) extends Partial[A,Nothing]
  case class Success[+B](get: B) extends Partial[Nothing,B]
  
  There is a type very similar to this called `Validation` in the Scalaz library. You can implement `map`, `map2`,
  `sequence`, and so on for this type in such a way that errors are accumulated when possible (`flatMap` is unable to
  accumulate errors--can you see why?). This idea can even be generalized further--we don't need to accumulate failing
  values into a list; we can accumulate values using any user-supplied binary function.
  
  It's also possible to use `Either[List[E],_]` directly to accumulate errors, using different implementations of
  helper functions like `map2` and `sequence`.
```

---

## 4.5 まとめ

* 関数型のエラー処理の基本原理を紹介した。
* さらに、高階関数を使ってエラーの処理と伝搬に共通するパターンをカプセル化できるようにすることを紹介した

---

## おわり
