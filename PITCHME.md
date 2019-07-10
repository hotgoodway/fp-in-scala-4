# fp-in-scala-4

例外を使わないエラー処理

---

## 例外の光と影

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

おわり
