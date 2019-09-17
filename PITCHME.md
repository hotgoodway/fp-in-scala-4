# fp-in-scala-8

パーサーコンビネータ
(parser combinator)

---

### パーサーとは

> テキスト、または記号、数字、トークンのストリームといった非構造化データを入力として受け取り、
そのデータを構造化して出力する、特殊なプログラムのこと。

---

## 9.1 代数の設計から始める

### 代数とは

> １つ以上のデータ型を操作する関数の集まりと、そうした関数の関係を指定する一連の法則のこと。

---

### 代数的設計

1. 法則が含まれている代数から作業を開始
2. 表現を後から決定する

**構文解析には特に適している**

---

### ライブラリーの設計目標

1. 表現力
2. 速度
3. 適切なエラー報告

---

### 最初は単純なパーサーから

```
def char(c: Char): Parser[Char]

```

Parser という型を作り出し、Parser の結果型を指定するパラメータを１つ定義している。

---

### パーサーの実行（代数を拡張）

* 成功した場合は有効な型の結果を返し
* 失敗した場合は失敗に関する情報を返し

```
def run[A](p: Parser[A])(input: String): Either[ParseError, A]
```

---

### traitを使って明示化

```
trait Parsers[ParseError, Parser[+_]]
  def run[A](p: Parser[A])(input: String): Either[ParseError, A]
  def char(c: Char): Parser[Char]
```

1. Parser は型パラメータであり、それ自体が共変の型コンストラクタである
2. Parser 型コンストラクタが Char に適用される

---

### 明白な法則

```
run(char(c))(c.toString) == Right(c)
```

---

### 文字列全体を認識する方法がないので追加

```
def string(s: String): Parser[String]
```

### 明白な法則

```
def run(string(s))(s) == Right(s)
```

---

### どちらかの文字列を認識したい場合

```
def orString(s1: String, s2: String): Parser[String]
```

### 多相にすると

```
def or[A](s1: Parser[A], s2: Parser[A]): Parser[A]
```

例:

```
run(or(string("abra"), string("cadabra")))("abra") == Rigth("abra")
run(or(string("abra"), string("cadabra")))("cadabra") == Rigth("cadabra")
```
