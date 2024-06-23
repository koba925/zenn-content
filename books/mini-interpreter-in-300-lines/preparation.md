---
title: "旅の準備"
---


## テスト

これから機能を追加していくことになります。機能を追加したら既存の機能が動かなくなった、ということがないようにテストを書いていきます。

複数回`print`することも出てきますのでリストに覚えておくようにします。

```diff py
 class Evaluator:
+    def __init__(self):
+        self._output = []
+
+    def clear_output(self): self._output = []
+    def output(self): return self._output
```

```diff py
     def evaluate(self, ast):
         match ast:
-            case ["print", val]: print(val)
+            case ["print", val]: self._output.append(val)
             case _: assert False, "Internal Error."
```

```diff py
if __name__ == "__main__":
    evaluator = Evaluator()
    while source := "\n".join(iter(lambda: input(": "), "")):
        try:
            ast = Parser(source).parse_program()
            print(ast)
            evaluator.clear_output()
            evaluator.eval_statement(ast)
            print(*evaluator.output(), sep="\n")
        except AssertionError as e: print(e)
```

テスト（新規作成）

Pythonについてくるunittestをフレームワークとして使います。
シンプルですがminilangのテストには十分な機能を持ってます。
ていうかフレームワークもいらないくらいなんですが、使っておくとvscodeの拡張機能が活用できて便利なので。

```py
import unittest

from minilang import Parser, Evaluator

def get_output(source):
    evaluator = Evaluator()
    evaluator.evaluate(Parser(source).parse())
    return evaluator.output()

def get_error(source):
    try: output = get_output(source)
    except AssertionError as e: return str(e)
    else: return f"Error not occurred. out={output}"

class TestMinilang(unittest.TestCase):
    def test_print(self):
        self.assertEqual(get_output("print 1;"), [1])
        self.assertEqual(get_output("  print\n  12  ;\n  "), [12])
        self.assertEqual(get_error("prin 1;"), "`print` expected, found `prin`.")
        self.assertEqual(get_error("print a;"), "Number expected, found `a`.")
        self.assertEqual(get_error("print 1:"), "Expected `;`, found `:`.")
        self.assertEqual(get_error("print 1"), "Expected `;`, found `$EOF`.")
        self.assertEqual(get_error("print 1;a"), "Expected `$EOF`, found `a`.")

if __name__ == '__main__':
    unittest.main()
```

数値の0とか1とかは何かのデフォルト値になってて意識せずにそういう値になってることもあるので避けてます。

テストはvscodeのテストエクスプローラで実施してもいいですし、unittestでやっても直接実行してもOKです。お好きなやり方で。
テストファイルはひとつしか使わない予定です。

```
$ python -m unittest
.
----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
$ python test_minilang.py 
.
----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

ちょっと修正してはテストする、を繰り返すには`Ctrl + :` `A`が便利です。


githubでは

## 少しリファクタリング

テストもできたのでちょっとリファクタリングします。

文法の決めごととコードとの対応がより明確になるようにします。
意外と機械的に書けていることにお気づきでしょう。

すでに100行を超えてますが、まだ整数をprintすることしかできてません。
