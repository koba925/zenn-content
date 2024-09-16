---
title: "制御構造"
---

制御構造を導入します。
制御構造を導入すると、ちょっとプログラムっぽいことが書けるようになります。
おたのしみに！

## if文を実装する

まずelse抜きのif文から実装します。一歩ずつ。

<if文>は`if <条件節> <結果節>`という形とします。
<条件節>は式、<結果節>は複文で、つまり`if <式> { ... }`という形になります。
CやJavascript風ではなくてGoやRust風です[^if-block]。

[^if-block]: then節（条件が真だった時に実行される文、つまり上記の`{ ... }`）やelse節で必ずスコープを作るのが狙いです。
`if (a = 1) var b = 1;`といったコードを許すと、`b`は宣言されてるのされてないの？という問題が発生するためそうしました。
書いたとおりに評価して、宣言されてない変数が使われたところでエラーにしてもよいのですが。

このやりかたの難点は `else if` と書けないところ。
`else { if ...`と書くのではインデントが深くなってしまいます。
minilangでは後ほど`elif`を導入して解決します。

ASTは`["if", <条件節>, <結果節>]`です。

Parserクラスから修正します。
statementにif文を追加します。

```diff py
     def _parse_statement(self):
         match self._current_token:
             case "{": return self._parse_block()
             case "var" | "set": return self._parse_var_set()
+            case "if": return self._parse_if()
             case "print": return self._parse_print()
             case unexpected: assert False, f"Unexpected token `{unexpected}`."
```

ここはもう問題ないでしょう。

if文の構文解析本体は以下の通りです。

```diff py
     def _parse_var_set(self):
         ...
 
+    def _parse_if(self):
+        self._next_token()
+        cond = self._parse_expression()
+        self._check_token("{")
+        conseq = self._parse_block()
+        return ["if", cond, conseq]
```

`if`の次に<式>を読み込んだら、その次が複文であること（`{`で始まること）を確認してブロックを読み込みます。

条件節はcondition、結果節はconsequenceと呼ばれることが多いのでそれにしたがって名前を付けています。
話のついでに、elseにあたるところは代替節（alternative）と呼ばれます。後で出てきます。
`then_clause`と`else_clause`とかにしてもよかった気がしますが、よく見かけるので豆知識を得たと思っていただければ。


`Evaluator`の修正です。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
             case ["set", name, value]: self._eval_set(name, value)
+            case ["if", cond, conseq]: self._eval_if(cond, conseq)
             ... 
     ...

     def _eval_set(self, name, value):
         ...
 
+    def _eval_if(self, cond, conseq):
+        if self._eval_expr(cond):
+            self._eval_statement(conseq)
```

条件節（＝`cond`）は式なので、まずここを評価して値が真またはPythonが真とみなす値であれば結果節（＝`conseq`）を評価します。
難しいことはしていません。
大事なことは条件節が真であることがわかるまでは結果節は評価しないということです。
当たり前に聞こえるかもしれませんが大事です。

実行します。

```
Input source and enter Ctrl+D:
if 5 = 5 { print 6; }
Output:
['program', ['if', ['=', 5, 5], ['block', ['print', 6]]]]
6
Input source and enter Ctrl+D:
if 5 # 5 { print 6; }
Output:
['program', ['if', ['#', 5, 5], ['block', ['print', 6]]]]

Input source and enter Ctrl+D:
if true { if true { print 6; } }
Output:
['program', ['if', True, ['block', ['if', True, ['block', ['print', 6]]]]]]
6
Input source and enter Ctrl+D:
if true { if false { print 6; } }
Output:
['program', ['if', True, ['block', ['if', False, ['block', ['print', 6]]]]]]

Input source and enter Ctrl+D:
if true print 5;
Output:
Error: Expected `{`, found `print`.
```

条件節の真偽で結果節が評価されたりされなかったりすることが確認できました。
ネストしたif文もちゃんと処理できています。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/7010_if) [差分](https://github.com/koba925/minilang-book/compare/6020_scope...7010_if)

## else節を実装する

結果節を読んだ後、次に来るのが`else`だったら代替節の処理を行います。

```diff py
     def _parse_if(self):
         self._next_token()
         cond = self._parse_expression()
         self._check_token("{")
         conseq = self._parse_block()
-        return ["if", cond, conseq]
+        alt = ["block"]
+        if self._current_token == "else":
+            self._next_token()
+            self._check_token("{")
+            alt = self._parse_block()
+        return ["if", cond, conseq, alt]
```

`alt`の初期値の`["block"]`は<文>が0個入った複文です。else節が存在しないときはそのままASTの<代替節>となり、「何もしない」ことを表します。

`Evaluator`では単純に代替節の処理を追加しているだけです。
代替節のないif文でも`alt`に`["block"]`を入れるようにしたため、代替節のありなしで場合を分けて処理する必要はありません。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
-            case ["if", cond, conseq]: self._eval_if(cond, conseq)
+            case ["if", cond, conseq, alt]: self._eval_if(cond, conseq, alt)
             ...

     ...

-    def _eval_if(self, cond, conseq):
+    def _eval_if(self, cond, conseq, alt):
         if self._eval_expr(cond):
             self._eval_statement(conseq)
+        else:
+            self._eval_statement(alt)
```

当然といえば当然なのですが、実装したいものと、それを実装するコードがそっくりなのが面白いところです。

動かします。

```
Input source and enter Ctrl+D:
if 5 = 5 { print 6; } else { print 7; }
Output:
['program', ['if', ['=', 5, 5], ['block', ['print', 6]], ['block', ['print', 7]]]]
6
Input source and enter Ctrl+D:
if 5 # 5 { print 6; } else { print 7; }
Output:
['program', ['if', ['#', 5, 5], ['block', ['print', 6]], ['block', ['print', 7]]]]
7
Input source and enter Ctrl+D:
if false { print 6; } else { if true { print 7; } else {print 8;} }
Output:
['program', ['if', False, ['block', ['print', 6]], ['block', ['if', True, ['block', ['print', 7]], ['block', ['print', 8]]]]]]
7
Input source and enter Ctrl+D:
if false { print 6; } else { if false { print 7; } else {print 8;} }
Output:
['program', ['if', False, ['block', ['print', 6]], ['block', ['if', False, ['block', ['print', 7]], ['block', ['print', 8]]]]]]
8
Input source and enter Ctrl+D:
if true { print 5; } else print 6;
Output:
Error: Expected `{`, found `print`.
```

GitHub: [コード](https://github.com/koba925/minilang-book/tree/7020_else) 差分

## elif

elifは今までのような`Parser`で構文を追加して`Evaluator`に新しいASTの評価を追加する、というのとは少し違ったやり方で作ります。

`if A { B } elif C { D } else { E } }`というのは`if A { B } else { if C { D } else { E } }`と同じで、後者はもうASTを渡してやれば評価できるようになっていますので、前者を見たら後者のASTと同じものを作ればよい、と考えます。

```diff py
     def _parse_if(self):
         self._next_token()
         cond = self._parse_expression()
         self._check_token("{")
         conseq = self._parse_block()
         alt = ["block"]
-        if self._current_token == "else":
+        if self._current_token == "elif":
+            alt = self._parse_if()
+        elif self._current_token == "else":
             self._next_token()
             self._check_token("{")
             alt = self._parse_block()
         return ["if", cond, conseq, alt]
```

`else`が来る位置で`elif`を見つけたら、その続きがif文とみなして構文解析し、その結果を代替節のASTとしています。
`elif C { D } else { E }`という形を`else { if C { D } else { E } }`と読み替えているわけです。

これだけ？
これだけです。
できあがるASTは同じものなので`Evaluator`は変更不要。

```
Input source and enter Ctrl+D:
if 5 # 5 { print 5; } elif 5 = 5 { print 6; } else { print 7; }
Output:
['program', ['if', ['#', 5, 5], ['block', ['print', 5]], ['if', ['=', 5, 5], ['block', ['print', 6]], ['block', ['print', 7]]]]]
6
Input source and enter Ctrl+D:
if 5 # 5 { print 5; } elif 5 # 5 { print 6; } else { print 7; }
Output:
['program', ['if', ['#', 5, 5], ['block', ['print', 5]], ['if', ['#', 5, 5], ['block', ['print', 6]], ['block', ['print', 7]]]]]
7
```

`else { if`で書いたときと同様のASTが出力されていることを見ておきましょう。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/7030_elif) [差分](https://github.com/koba925/minilang-book/compare/7020_else...7030_elif)

## while

ifができればwhileも簡単です[*while-easy]。

[*while-easy]: breakやcontinueを作らなければ。

<while文>は`while <条件節> <本体>`という形とします。
<条件節>は式、<本体>は複文で、つまり`while <式> { ... }`という形になります。
elseなしのif文とまったく同じ構文ですね。

なので構文解析はelseなしif文の構文解析と全く同じ構造になります。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
             case "if": return self._parse_if()
+            case "while": return self._parse_while()
             case "print": return self._parse_print()
             ...
     ...

     def _parse_if(self):
         ...

+    def _parse_while(self):
+        self._next_token()
+        cond = self._parse_expression()
+        self._check_token("{")
+        body = self._parse_block()
+        return ["while", cond, body]
```

評価も簡単。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
             case ["if", cond, conseq, alt]: self._eval_if(cond, conseq, alt)
+            case ["while", cond, body]: self._eval_while(cond, body)
             case ["print", expr]: self._eval_print(expr)
             ...
 
     ...

     def _eval_if(self, cond, conseq, alt):
         ...
 
+    def _eval_while(self, cond, body):
+        while self._eval_expr(cond):
+            self._eval_statement(body)
```

やっぱりwhileはwhileで実装します。

動かします。

```
Input source and enter Ctrl+D:
var i = 0;
while i # 3 {
    print i;
    set i = i + 1;
}
Output:
['program', ['var', 'i', 0], ['while', ['#', 'i', 3], ['block', ['print', 'i'], ['set', 'i', ['+', 'i', 1]]]]]
0
1
2
```

繰り返しができるようになるとコンピュータっぽい？仕事ができるようになります。
フィボナッチ数列とか。

```
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

かなりプログラミング言語を作ってる感じがしてきたのではないでしょうか？

ところで、ASTの表示がかなり長くなってきました。
ちょっとこのままでは見る気がしませんね。

この出力はPythonとして評価できる形なので、エディタにコピペしてPythonとしてフォーマットをかけてやると見やすくなります[^pprint]。
手元の環境(vscode＋ごく普通のExtension)ではこうなりました。
これくらいならちょっと確認しておこう、くらいの気持ちになれるのではないでしょうか？

[^pprint]: はじめから見やすく表示したい！と思ったら`pprint.pprint()`をお試しください。
似たような表示になります（本書では行数が増えてしまうので採用しませんでした）。

```py
[
    "program",
    ["var", "i", 0],
    ["var", "a", 1],
    ["var", "b", 0],
    ["var", "tmp", 0],
    [
        "while",
        ["#", "i", 5],
        [
            "block",
            ["print", "a"],
            ["set", "tmp", "a"],
            ["set", "a", ["+", "a", "b"]],
            ["set", "b", "tmp"],
            ["set", "i", ["+", "i", 1]],
        ],
    ],
]
```

今後、ASTの表示が長いときは適宜省略します。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/7040_while) [差分](https://github.com/koba925/minilang-book/compare/7030_elif...7040_while)