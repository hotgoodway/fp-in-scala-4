# fp-in-scala-4

例外を使わないエラー処理

---

## 例外の光と影

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

1. 例外ベースの複雑なコードが記述可能

昔の忠告: 例外はエラー処理のみ使用するべきであり、制御フローには使用するべきではない

---

### 2つ問題

2. 例外が型安全ではない

failingFn, Int => Intの型から例外が発生することなど何もわからないので実行時まで検出できません

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


