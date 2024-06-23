---
title: "複数の文を実行できるようにする"
---

●はじめからできるようにすっかね
●blockはifの準備に入れる（スコープを関数のところでやｒｙなら

ここまではprint文を1回実行するだけでした。
複数の文を実行できるようにします。

`print 1; print 2;`のような入力ソースコードを受け取ったら、構文解析では`["block", ["print", 1], ["print", 1]]`のようなASTを作り、評価では`block`の中身をひとつずつ実行するようにします。
`block`もひとつの文として扱います。複数の文をまとめた文、ということです。

トークンのグループの先頭を指している状態にしておくことにします。
これから関数が増えてくると、なにかしら統一しておかないとわかりづらくなるので。
型を指定しているのは警告除けです。なくても動きます。


```diff py
     def __init__(self, source):
         self.scanner = Scanner(source)
         self.current_token = ""
+        self._next_token()
```

diffがちょっとわかりづらいかもしれません。
作業的には、元の`parse()`を`_print()`に変更して`parse()`に新しいコードを書いている感じです。`_print()`の中身も影響を受けて修正が入ります。今後こういうパターンがよく出てくると思いますので慣れてください（他力本願）。

```diff py 
     def parse(self):
-        command = self._next_token()
-        assert command == "print", f"`print` expected, found `{command}`."
+        block: list = ["block"]
+        while self.current_token != "$EOF":
+            block.append(self._print())
+        return block
```

```diff py
+    def _print(self):
+        assert self.current_token == "print", f"`print` expected, found `{self.current_token}`."
 
        number = self._next_token()
        assert isinstance(number, int), f"Number expected, found `{number}`."

        self._next_token()
        self._consume(";")
 
-        self._consume("$EOF")
-
         return ["print", number]
```

matchのパターンマッチング超便利。

```diff py
     def evaluate(self, ast):
         match ast:
+            case ["block", *statements]: self._block(statements)
             case ["print", val]: self._output.append(val)
             case _: assert False, "Internal Error."
```

```diff py 
+    def _block(self, statements):
+        for statement in statements:
+            self.evaluate(statement)
```

軽く動かしてみます。

```
$ python minilang.py 
: print 1; print 2;
: 
1
2
:
```

動きました。
エラーの出方が変わるところを削除して新たなテストを追加。
今回の変更で、からっぽの入力でもOKなことになったのでそういうテストも入れておきます（そんな仕様はおかしい！と思ったらエラーになるようチェックを入れてみてください）。

```diff py
         self.assertEqual(get_error("print a;"), "Number expected, found `a`.")
         self.assertEqual(get_error("print 1:"), "Expected `;`, found `:`.")
         self.assertEqual(get_error("print 1"), "Expected `;`, found `$EOF`.")
-        self.assertEqual(get_error("print 1;a"), "Expected `$EOF`, found `a`.")
+
+    def test_statements(self):
+        self.assertEqual(get_output(""), [])
+        self.assertEqual(get_output("print 1; print 2;"), [1, 2])
+        self.assertEqual(get_error("print 1; prin"), "`print` expected, found `prin`.")
```

## ブロックを実装する

`{}`のことです。
この時点でやってもあまり面白味はないんですが、`{}`の中身の処理は全く同じなのでここでやっておきます。
さっきは入力ソースコードの最初から最後まで実行していたのが、`{`から`}`まで実行するようになっただけ。

文が2種類になったことの影響の方が大きいです。
print文と複文(block)を合わせて文(statement)として扱います。

プログラム全体は、statementの集まりです。

```diff py
     def parse(self):
         block: list = ["block"]
         while self.current_token != "$EOF":
-            block.append(self._print())
+            block.append(self._statement())
         return block
```

```diff py
+    def _statement(self):
+        match self.current_token:
+            case "{": return self._block()
+            case "print": return self._print()
```

```diff py 
+    def _block(self):
+        block: list = ["block"]
+        self._next_token()
+        while self.current_token != "}":
+            block.append(self._statement())
+        self._next_token()
+        return block
```

```diff py
     def _print(self):
-        assert self.current_token == "print", f"`print` expected, found `{self.current_token}`."
         number = self._next_token()
```

再帰下降パーザが最上位の要素から「下降」している感じ、すこし感じられたでしょうか？

動かしてみます。

```
$ python minilang.py 
: print 1; { print 2; print 3; } print 4;
: 
1
2
3
4
: 
```

大丈夫そうですね。

ただ、これだけだとほんとに複文として処理してるのか？というのが見えづらい

`Evaluator.evaluate()`でデバッガを使うなり`print`を入れてみたりすれば、`['block', ['print', 1], ['block', ['print', 2], ['print', 3]], ['print', 4]]`というASTが渡されていることがわかるはずです。



テストの更新はこちら。

```diff py
     def test_print(self):
         self.assertEqual(get_output("print 1;"), [1])
         self.assertEqual(get_output("  print\n  12  ;\n  "), [12])
-        self.assertEqual(get_error("prin 1;"), "`print` expected, found `prin`.")
+        self.assertEqual(get_error("prin 1;"), "Unexpected token `prin`.")
         self.assertEqual(get_error("print a;"), "Number expected, found `a`.")
         self.assertEqual(get_error("print 1:"), "Expected `;`, found `:`.")
         self.assertEqual(get_error("print 1"), "Expected `;`, found `$EOF`.")
```

```diff py
     def test_statements(self):
         self.assertEqual(get_output(""), [])
         self.assertEqual(get_output("print 1; print 2;"), [1, 2])
-        self.assertEqual(get_error("print 1; prin"), "`print` expected, found `prin`.")
+        self.assertEqual(get_error("print 1; prin"), "Unexpected token `prin`.")
```

```diff py
+    def test_block(self):
+        self.assertEqual(Parser("print 1; { print 2; print 3; } print 4;").parse(),
+                         ["block", ["print", 1], ["block", ["print", 2], ["print", 3]], ["print", 4]])
+        self.assertEqual(get_output("print 1; { print 2; print 3; } print 4;"), [1, 2, 3, 4])
 
 if __name__ == "__main__":
```

すべての差分はこちら。
https://github.com/koba925/minilang/compare/statements...block
