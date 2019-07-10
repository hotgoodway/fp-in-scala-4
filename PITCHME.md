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

```
def mean(xs: Seq[Double]): Option[Double] =
  if (xs.isEmpty) None
  elase Some(xs.sum / xs.length)
```

完全な関数になる

---

### 4.3.1 Option の使用パターン

- 指定されたキーによるマップ検索
- リストなどで定義されている headOption と lastOption
...

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

### EXECISE 4.1

基本関数を全て実装せよ

