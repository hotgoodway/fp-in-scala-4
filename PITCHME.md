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

---

### 結論

yが参照透過ではない!

---


おわり
