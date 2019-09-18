# fp-in-scala-8

パーサーコンビネータ
(parser combinator)

---

### パーサーとは

> テキスト、または記号、数字、トークンのストリームといった非構造化データを入力として受け取り、
そのデータを構造化して出力する、特殊なプログラムのこと。

---

## 9.1 代数の設計から始める

#### 代数とは

> １つ以上のデータ型を操作する関数の集まりと、そうした関数の関係を指定する一連の法則のこと。

---

#### 代数的設計

1. 法則が含まれている代数から作業を開始
2. 表現を後から決定する


**構文解析には特に適している**

---

#### ライブラリーの設計目標

1. 表現力
2. 速度
3. 適切なエラー報告

---

#### 最初は単純なパーサーから

```
def char(c: Char): Parser[Char]

```

Parser という型を作り出し、

Parser の結果型を指定するパラメータを１つ定義している。

---

#### パーサーの実行（代数を拡張）

* 成功した場合は有効な型の結果を返し
* 失敗した場合は失敗に関する情報を返し


```
def run[A](p: Parser[A])(input: String): Either[ParseError, A]
```

---

#### traitを使って明示化

```
trait Parsers[ParseError, Parser[+_]]
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]
  def char(c: Char): Parser[Char]
```

1. Parser は型パラメータであり、それ自体が共変の型コンストラクタである
2. Parser 型コンストラクタが Char に適用される

#### 明白な法則

```
run(char(c))(c.toString) == Right(c)
```

---

#### 文字列全体を認識する方法がないので追加

```
def string(s: String): Parser[String]
```

#### 明白な法則

```
def run(string(s))(s) == Right(s)
```

---

#### どちらかの文字列を認識したい場合

```
def orString(s1: String, s2: String): Parser[String]
```


#### 多相にすると

```
def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]
```

例:

```
run(or(string("abra"), string("cadabra")))("abra") == Rigth("abra")
run(or(string("abra"), string("cadabra")))("cadabra") == Rigth("cadabra")
```

---

#### パーサーへの中置構文

```
trait Parsers[ParseError, Parser[+_]] { self =>
  ...
  def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]
  implicit def string(s: String): Parser[String]
  implicit def operators[A](p: Parser[A]) = ParserOps[A](p)
  
  // StringがaParserへ自動的に昇格される
  implicit def asStringParser[A](a: A)(implicit f: A => Parser[String]):
    ParserOps[String] = ParserOps(f(a))
  
  case class ParserOps[A](p: Parser[A]) {
    // or(a, b)
    def |[B>:A](p2: Parser[B]): Parser[B] = self.or(p, p2)
    def or[B>:A](p2: => Parser[B]): Parser[B] = self.or(p, p2)
  }
}
```

---

#### 繰り返し文字列の認識

```
def listOfN[A](n: Int, p: Parser[A])]: Parser[List[A]]
```

例:

```
run(listOfN(3, "ab" | "cad"))("ababcad") == Right("ababcad")
run(listOfN(3, "ab" | "cad"))("cadabab") == Right("cadabab ")
run(listOfN(3, "ab" | "cad"))("ababab") == Right("ababab")
```

---

#### 課題

1. 必要なコンビネータが揃ったが、代数を最小限のプリミティブに絞り込むことができていない
2. より汎用的な法則についても語られていない

---

#### ガイドラインとなる質問

1. 'a'の文字を 0 個以上認識する Parser[Int]
2. 'a'の文字を 1 個以上認識する Parser[Int]、失敗（見つからない）した場合のヒント
3. 0 個以上の'a'に続いて 1個以上の'b'を認識するパーサー
4. 長さを取り出して削除するためだけにList[Char]の構築が効率悪いためどう解決?
5. 様々な形式の繰り返しは本書の代数のプリミティブなの?
6. ...
7. a | b は b | a と同じ?
8. a | (b | c) は (a | b) | c と同じ?
9. ...

---

## 9.2 代数の例

コンビネータを洗い出してみよう。

---

```
// 文字 'a' の 0個以上の繰り返しを認識し、検出するためのプリミティブコンビネータ
def many[A](p: Parser[A]): Parser[List[A]]

// 必要なのは要素の数を数えるためコンビネータmapを追加
def map[A, B](a: Parser[A])(f: A => B): Parser[B]

// 合成したら、以下のように Parser が定義できる
map(many(char('a')))(_.size)

// 便利な構文
val numA: Parser[Int] = char('a').many.map(_.size)
```

---

#### Parser と map の結合

```
trait Parsers[ParseError, Parser[+_]] {
  ...
  
  // Prop を使って法則を実行可能になった
  object Laws {
    def equal[A](p1: Parser[A], p2: Parse[A])(in: Gen[String]): Prop =
      forAll(in)(s => run(p1)(s) == run(p2)(s))
      
    def mapLaw[A](p: Parse[A])(in: Gen[String]): Prop =
      equal(p, p.map(a => a))(in)
  }
 
}
```

---

mapが利用できたので、stringに基づいてcharの実装

```
def char(c: Char): Parser[Char] =
  string(c.toString) map (_.charAt(0))
```

---

string と mapを使って succeedも定義可能

```
// string("")は任意な入力があっても常に成功するため、
// このパーサーは入力文字列に関係なく常に a の値で成功する
def succeed[A](a: A): Parser[A] =
  string("") map (_ => a)
```

---

### 9.2.1 スライスと空ではない繰り返し

many と map を組み合わせて、文字'a'の数を数えるのが、
List[Char]を構築するのが効率悪いので、改善のコンビネータを作成。

```
def slice[A](p: Parser[A]): Parser[String]
```

???

---

文字列'a'を１つ以上認識したいための新しいコンビネータを定義

```
def many1[A](p: Parser[A]): Parser[List[A]]
```







