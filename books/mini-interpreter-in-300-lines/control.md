---
title: "制御構造"
---

## if文を実装する

まずelse抜きのif文から実装します。一歩ずつ。

minilangのif文は`if <数> { ... }`という形にします。つまりCやJavascript風ではなくてGoやRust風にします。これは秘密ですがthen節（条件が真だった時に実行される文、つまり上記の`{ ... }`）やelse節が複文でなく単文だった場合、後ほどスコープを導入するときにすこし面倒があるので[^if-block]。

[^if-block]: このやりかたの難点は `else if` と書けないところ。`if`を特別扱いするか、`elif`を導入するなどする必要あり。minilangでは`else { if ...`と書いていただきます。インデントが深くなりますがご了承ください（または自分で実装してみてください）。

条件にあたるところが<数>なのはまだminilangが式を扱えないからです。真偽値もないので0を偽、0以外を真として扱います。
つまり<数>が0以外のときだけ`{ ... }`を実行します。

Parserクラスから修正します。
statementにif文を追加します。

```diff py
     def _statement(self):
         match self.current_token:
             case "{": return self._block()
+            case "if": return self._if()
             case "print": return self._print()
             case token: assert False, f"Unexpected token `{token}`."
```

if <condition> then <consequence> else <alternative> のような構文で、<condition>の部分を日本語では条件節と呼びますが、<consequence> や <alternative> のことは日本語で何と言いますか？

条件節、結果節（帰結節）、代替節などと訳されているようです。
then_clauseとelse_clauseとかにしてもよかった気がしますが、この言い方も見かけるので豆知識を得たと思ってください。

if文の処理本体は以下の通りです。
`if`の次が条件（数）で、そのつぎが複文であることを確認します。
`_block()`は`_current_token`が`{`を指した状態で呼ばなければいけないので、`self._consume("{")`とすることができず、`_current_token`を進めない`_check()`を作りました。

```diff py
+    def _if(self):
+        condtion = self._next_token()
+        assert isinstance(condtion, int), f"Number expected, found `{condtion}`."
+        self._next_token()
+        self._check("{")
+        consequence = self._block()
+        return ["if", condtion, consequence]
```

```diff py
-    def _consume(self, expected_token):
+    def _check(self, expected_token):
         assert self.current_token == expected_token, \
                f"Expected `{expected_token}`, found `{self.current_token}`."
+
+    def _consume(self, expected_token):
+        self._check(expected_token)
         self._next_token()
```

```
$ python minilang.py 
: if 1 { print 2; }
: 
2
: if 0 { print 2; }
: 

:
```

ほかのテストには影響を与えないのでテスト項目は追加のみです。

```
+    def test_if(self):
+        self.assertEqual(get_output("if 1 { print 2; }"), [2])
+        self.assertEqual(get_output("if 0 { print 2; }"), [])
+        self.assertEqual(get_output("if 1 { if 1 { print 2; } }"), [2])
+        self.assertEqual(get_output("if 1 { if 0 { print 2; } }"), [])
+        self.assertEqual(get_error("if a { print 2; }"), "Number expected, found `a`.")
+        self.assertEqual(get_error("if 1 print 2;"), "Expected `{`, found `print`.")
```


## else節を実装する

then節(`consequence`)を読んだ後、次に来るのが`else`だったらelse節(`alternative`)の処理を行います。
`["block"]`はstatementが0個入った複文です。else節が存在しないとき、「何もしない」ことを表すために渡しています。

実装したいものと、それを実装するコードがそっくりなのが面白い
当然といえば当然

```diff py
         self._next_token()
         self._check("{")
         consequence = self._block()
-        return ["if", condtion, consequence]
+        alternative = ["block"]
+        if self._current_token == "else":
+            self._next_token()
+            self._check("{")
+            alternative = self._block()
+        return ["if", condtion, consequence, alternative]
```

```diff py
     def evaluate(self, ast):
         match ast:
             case ["block", *statements]: self._block(statements)
-            case ["if", condition, consequence]: self._if(condition, consequence)
+            case ["if", condition, consequence, alternative]:
+                self._if(condition, consequence, alternative)
             case ["print", val]: self._output.append(val)
             case _: assert False, "Internal Error."
```

```diff py
-    def _if(self, condition, consequence):
-        if condition != 0: self.evaluate(consequence)
+    def _if(self, condition, consequence, alternative):
+        if condition != 0:
+            self.evaluate(consequence)
+        else:
+            self.evaluate(alternative)
```

```
: if 1 { print 2; } else { print 3; }
: 
2
: if 0 { print 2; } else { print 3; }
: 
3
: 
```

```diff py
+    def test_else(self):
+        self.assertEqual(get_output("if 1 { print 2; } else { print 3; }"), [2])
+        self.assertEqual(get_output("if 0 { print 2; } else { print 3; }"), [3])
+        self.assertEqual(get_error("if 1 { print 2; } else print 3;"), "Expected `{`, found `print`.")
```

そろそろパターンが見えてきた感じがしましたか？

## elif

if文の構文解析にちょっとだけ追加します。`Evaluator`は変更不要。
気が付けば意外なほど瞬殺です。

## while

ifができればwhileは瞬殺です[*while-easy]。

[*while-easy]: breakやcontinueを作らなければ。

```
$ python minilang.py
Input source and enter Ctrl+D:
var i = 0; var a = 1; var b = 0; var tmp = 0;
while i # 5 {
    print a;
    set tmp = a; set a = a + b; set b = tmp;
    set i = i + 1;
}
Output:
['program', ['var', 'i', 0], ['var', 'a', 1], ['var', 'b', 0], ['var', 'tmp', 0], ['while', ['#', 'i', 5], ['block', ['print', 'a'], ['set', 'tmp', 'a'], ['set', 'a', ['+', 'a', 'b']], ['set', 'b', 'tmp'], ['set', 'i', ['+', 'i', 1]]]]]
1
1
2
3
5
```
