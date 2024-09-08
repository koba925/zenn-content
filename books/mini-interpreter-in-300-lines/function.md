---
title: "関数"
---

いよいよ関数です。関数を実装するとminilangの完成です。
いろいろ実装することがあるので長い章になります。
一度に実装するとできるだけ細切れにしていきます。

## 組み込み関数と関数呼び出し

関数の導入にあたって、スモールステップで導入するため自分で関数を定義するのは後回しにして、まず組み込み関数を作ります。
関数の呼び出しだけ実装すればよいので。
ここでは`less`という名前で `a < b` なら true、そうでなければfalseを返す組み込み関数を作ります[^less]。

[^less]: `<`を作ればいい話なんですができるだけシンプルなものを、と考えたらこうなりました。

組み込み関数はPythonの関数そのもので表すことにします。
あらかじめ環境に名前が`less`で値が`lambda a, b: a < b`という変数を作っておき、それを使うようにします。
数字でも文字列でもないので、「組み込み関数型」と言うべき型が増えたことになります。

関数の呼び出しの構文は`<関数>(<式>, <式>, ...)`、ASTは`[<関数>], <式>, <式>, ...]`とします。
`less`で言うと`less(1, 2)`、`["less", 1, 2]`などとなります。
ASTは組み込み関数であることを明示して`["builtin", "less", 1, 2]`などとしてもよいのですがそうしなくてもわかるので。

<関数>はたいていの場合識別子ですが、最終的に関数を返してくれる<式>であればなんでも使えます。

優先度は高めで、powerとprimaryの間です。
優先度と言ってもちょっとイメージづらいかもしれませんが、`less(1, 2)`で言うと左辺が`less`、演算子が`(`、右辺が`1,2`、最後のカッコは終わりを示すための単なるしるしなので無視、と思ってください。
そうすれば`+`などと同じように見えてきませんか？

ユーザ定義関数型が出てきてからの話になりますが、関数を返す関数を定義して`some_func_returns_func(2, 3)(4)`のように呼び出しを連続して書くこともできるようにします。
足し算を`2 + 3 + 4`と連続して書けるのと同じことです。
この場合、まず`some_func_returns_func(2, 3)`を評価して、その結果として返ってきた関数に`4`を引数として与えて呼ぶ、とうことになりますので、これは左結合です。
右結合だとするとまず`(2, 3)(4)`を評価して、ということになりますがこれでは意味が分かりません。

また、`less`の場合は引数の数はふたつですが、引数の数はいくつでもいいようにします。

```diff py
     def _parse_power(self):
-        power = self._parse_primary()
+        power = self._parse_call()
         if self._current_token != "^": return power
         self._next_token()
         return ["^", power, self._parse_power()]
 
+    def _parse_call(self):
+        call = self._parse_primary()
+        while self._current_token == "(":
+            self._next_token()
+            args = []
+            while self._current_token != ")":
+                args.append(self._parse_expression())
+                if self._current_token != ")":
+                    self._consume_token(",")
+            call = [call] + args
+            self._consume_token(")")
+        return call
```

`_parse_call`の全体の形は`_parse_binop_left`に似ています。
演算子（ここでは`(`のこと）の右辺（`<式>, <式>, ...`）を評価するところが内側の`while`です。
式を評価して、次のトークンが`,`ならば次の式を、`)`ならループを終了するようになっています。

次は評価です。
まず環境に変数`less`の値を`lambda a, b: a < b`と定義しておきます。

```diff py
     def __init__(self):
         self.output = []
         self._env = Environment()
+        self._env.define("less", lambda a, b: a < b)
```

`less`も`1`や`true`などと同様、普通に値です。
print文にも手を入れて、「組み込み関数型」も表示できるようにします。

```diff py
     def _to_print(self, value):
         match value:
             case bool(b): return "true" if b else "false"
+            case v if callable(v): return "<builtin>"
             case _: return value
```

表示は単に`<builtin>`としました。
`print less;`を実行すると`<builtin>`と表示されます。
`print less(1, 2);`とは違って、関数そのものをprintしていることに注意してください。

では実際に組み込み関数を評価するコードを追加します。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
             case ["#", a, b]: return self._eval_expr(a) != self._eval_expr(b)
+            case [func, *args]:
+                func = self._eval_expr(func)
+                args = [self._eval_expr(arg) for arg in args]
+                return func(*args)
             case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

`case [func, *args]:`でASTから関数部分と引数部分を取り出したら、`self._eval_expr(func)`で関数の実体を取得します。
`less`であれば`lambda a, b: a < b`が関数の実体です。
`[self._eval_expr(arg) for arg in args]`で引数の式をそれぞれ評価したらそれらを引数として関数の実体を呼び出します。
これを関数を引数に適用(apply)する、と言います。

ユーザ定義関数ではもっといろんなことをしますが、大枠は同じなので雰囲気をつかんでおいてください。

厳しくやりたい[^loose]ときはここで仮引数の数と実引数の数が合ってるか確認しますが、minilangでは省略します[^inspect]。

[^loose]: 厳しくない言語もあります。jsとかjsとかjsとか。


[^inspect]: 関数の引数について調べるならinspectモジュールを使うことができます。
たとえば`len(inspect.signature(lambda a, b: a + b).parameters)`は2を返します。
可変長引数とかキーワード引数もあるのでそう単純にはいかないときもありますが、そういうときは`lambda`で囲むとか。

実行します。

```
Input source and enter Ctrl+D:
print less(5 + 6, 5 * 6);
Output:
['program', ['print', ['less', ['+', 5, 6], ['*', 5, 6]]]]
true
Input source and enter Ctrl+D:
print less(5 * 6, 5 + 6);
Output:
['program', ['print', ['less', ['*', 5, 6], ['+', 5, 6]]]]
false
Input source and enter Ctrl+D:
print less;
Output:
['program', ['print', 'less']]
<builtin>
Input source and enter Ctrl+D:
print less(5 * 6 7);
Output:
Error: Expected `,`, found `7`.
```

`less`のおかげで互除法が書けます[^euclidean-algorithm]。

[^euclidean-algorithm]: minilangには剰余がないので引き算で実装しています。

```
Input source and enter Ctrl+D:
var a = 36; var b = 24; var tmp = 0;
while b # 0 {
    if less(a, b) {
        set tmp = a; set a = b; set b = tmp;
    }
    set a = a - b;
}
print a;
Output:
[...]
12
```

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8010_builtin_function) [差分](https://github.com/koba925/minilang-book/compare/7040_while...8010_builtin_function)

## 式文

関数の実装とは直接関係ないのですが、`print_sum(1, 2);`みたいなのが書けるようにしておきます。
`<式>;`という形です。式文(expression statement)と呼ばれます。

もしかすると当たり前すぎて何を言っているのかわからないかもしれませんが、実は現状のminilangではそういうのが書けないのです。
プログラムは文の集まりで、文にはprint文・if文・while文などしかなく、式はこれらの文の中でしか書けないので。

式文では、式を評価した結果は捨てられるだけなので`1 + 2;`などと書くことはできますが何も起こりません。
式文はいわゆる「副作用」を目的とした関数呼び出しのためにあります。
minilangで有効なのはprint文を含む関数を呼び出すときくらいです。

式の先頭に未知のトークンが出てきたとき、今まではエラーにしていましたが、きっと式文に違いないと想定して処理するようにします。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
             case "print": return self._parse_print()
-            case unexpected: assert False, f"Unexpected token `{unexpected}`."
+            case _: return self._parse_expression_statement()
```

構文解析は簡単です。
まず<式>を読んで、次にセミコロンが続いていることを確認するだけです。
式文のASTは`["expr", <式>]`です。

```diff py
     def _parse_print(self):
         ...
 
+    def _parse_expression_statement(self):
+        expr = self._parse_expression()
+        self._consume_token(";")
+        return ["expr", expr]

     def _parse_expression(self):
         ...
```

評価では式の部分を評価するだけです。
`_eval_expr()`がそのまま使えます。
戻り値は使わないで捨ててしまいます。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
             case ["print", expr]: self._eval_print(expr)
+            case ["expr", expr]: self._eval_expr(expr)
             case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

実行します・・・

```
Input source and enter Ctrl+D:
5 + 6;
Output:
['program', ['expr', ['+', 5, 6]]]

```

・・・が、エラーにならないだけでたいしたことは起きません。
ので、ちょっとステキな（そこまででもない）組み込み関数を追加します。

```diff py
 class Environment:
     ...
 
     def get(self, name):
         ...
 
+    def list(self):
+        if self._parent is None: return [self._values]
+        return self._parent.list() + [self._values]

 class Evaluator:
     def __init__(self):
         self.output = []
         self._env = Environment()
         self._env.define("less", lambda a, b: a < b)
+        self._env.define("print_env", self._print_env)
+
+    def _print_env(self):
+        for values in self._env.list():
+            print({ k: self._to_print(v) for k, v in values.items() })
```

`_print_env`はメソッドですが、`self._print_env`としてやることで関数として扱うことができます。

`print_env();`で呼んだ時の環境を表示してくれます。

```
Input source and enter Ctrl+D:
var a = 5; { var b = 6; { var c = 7; print_env(); } }
Output:
{'less': '<builtin>', 'print_env': '<builtin>', 'a': 5}
{'b': 6}
{'c': 7}
```

上にある方が親の環境です。最上位に組み込み関数もいるのが見えます。
イメージと合ってましたか？
少し試して、慣れておきましょう。

これから環境の動きがちょっと複雑になってわかりにくくなってきます。
なんか動きがおかしいなと思ったら呼んでみましょう。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8020_expression_statement) [差分](https://github.com/koba925/minilang-book/compare/8010_builtin_function...8020_expression_statement)

## ユーザ定義関数の追加

組み込み関数ができましたので、次はいよいよユーザ定義関数を書けるようにします。
組み込み関数と同じように、ひとつ型を追加する感じになります。
まずは「ユーザ定義関数型」を書けるようにしましょう。
呼び出すのは後で、ということです。

minilangでは、無名関数（Pythonで言うところの`lambda`）をベースに実装していきます。
`lambda a: print(a)`は`func(a) { print a; }`と書きます。
`def pr(a): print(a)`に当たる書き方は`var pr = func(a) { print a; }`です。
呼ぶときは普通に`pr(3);`などと呼べます。

ついでに用語の定義ですが、この`pr`を例にとると、定義時の引数（`a`）を仮引数（parameter）、呼ぶときの引数を実引数（argument）と呼びます。
どっちがどっちだか忘れそうになりますが、「仮」と「実」のイメージと結び付けて覚えてください。
仮引数がparameterであることは「パラメータ」という言葉の使い方からなんとなくイメージできる気がします。

JavaScriptやっているひとにはおなじみかもしれませんが、名前は必須ではなく、名前を付けずに呼ぶこともできます。
`func(a) { print a; }(3)`とすると、`func(a) { print a; }`という関数に`3`という引数を与えて実行します。
`func`・`()`・`{}`のつながりの方が`{}`と`(3)`の間のつながりよりも優先順位が高いということでもあります。
`func`は最後の`}`まで読むと値になるので、`primary`の仲間になります。

<関数>の構文は`func (<識別子>, <識別子>, ...) <複文>`で、ASTは`["func", ]
パーツが多いので、構文解析のコードがいつもよりすこし長めですが順番に見ていけばわかるはずです。

```diff py
     def _parse_primary(self):
         match self._current_token:
             case "(":
                 ...
+            case "func": return self._parse_func()
             case int(value) | bool(value) | str(value):
                 ...
 
+    def _parse_func(self):
+        self._next_token()
+        params = self._parse_parameters()
+        body = self._parse_block()
+        return ["func", params, body]
+
+    def _parse_parameters(self):
+        self._consume_token("(")
+        params = []
+        while self._current_token != ")":
+            param = self._current_token
+            assert isinstance(param, str), f"Name expected, found `{param}`."
+            self._next_token()
+            params.append(param)
+            if self._current_token != ")":
+                self._consume_token(",")
+        self._consume_token(")")
+        return params
```

「ユーザ定義関数型」が増えていますので、`_to_print`に1行追加しておきます。

```diff py
     def _to_print(self, value):
         match value:
             case bool(b): return "true" if b else "false"
             case v if callable(v): return "<builtin>"
+            case ["func", *_]: return "<func>"
             case _: return value
```

`["func", param, body]`というASTを受け取ったら、`["func", param, body]`というユーザ定義関数型の値を返します。
同じものを返していますがこれはある意味たまたまで、同じにしなければならないというわけではありません。
そういう形でユーザ関数型の値を表すことに決めた、というだけです。

ASTに`5`とあれば値の`5`を返していたのと同じです。
整数の5を`["int", 5]`で表すことにすることもできるけれども単に`5`で表すことにした、という決めの問題です。

```diff py
     def _eval_expr(self, expr):
         match expr:
             case int(value) | bool(value): return value
             case str(name): return self._eval_variable(name)
+            case ["func", param, body]: return ["func", param, body]
             case ["^", a, b]: return self._eval_expr(a) ** self._eval_expr(b)
             ...
```

呼び出しは次の節なのでこの節はここまでです。
一応動かしてみましょう。

```diff py
Input source and enter Ctrl+D:
func() {};
Output:
['program', ['expr', ['func', [], ['block']]]]

Input source and enter Ctrl+D:
print func() {};
Output:
['program', ['print', ['func', [], ['block']]]]
<func>
Input source and enter Ctrl+D:
func(a, b) { print a + b; };
Output:
['program', ['expr', ['func', ['a', 'b'], ['block', ['print', ['+', 'a', 'b']]]]]]

```

ASTがちゃんとできていることが確認できました。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8030_user_function_type) [差分](https://github.com/koba925/minilang-book/compare/8020_expression_statement...8030_user_function_type)

## ユーザ定義関数の呼び出し・実行

ユーザ定義関数の呼び出しを実装します。
わかってしまえば単純なしくみなんですが、初めて見るときはちょっと難しいかもしれません。
ここが一番のミソと言える部分ですのでがんばってください。

ユーザ定義関数の呼び出しの構文はは組み込み関数と同じなので、構文解析に変更はありません。
評価のみ変更します。
組み込み関数とユーザ定義関数は同じ形のASTなので、同一のパターンで処理します。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
             case ["#", a, b]: return self._eval_expr(a) != self._eval_expr(b)
             case [func, *args]:
-                func = self._eval_expr(func)
-                args = [self._eval_expr(arg) for arg in args]
-                return func(*args)
+                return self._apply(self._eval_expr(func),
+                                   [self._eval_expr(arg) for arg in args])
             case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

関数と引数はここで評価を済ませてしまい、残りの処理を`_apply()`で行います。

```diff py
     def _div(self, a, b):
         ...
 
+    def _apply(self, func, args):
+        if callable(func): return func(*args)
+
+        [_, parameters, body] = func
+        parent_env = self._env
+        self._env = Environment(parent_env)
+        for param, arg in zip(parameters, args): self._env.define(param, arg)
+        self._eval_statement(body)
+        self._env = parent_env
+        return 0
```

`func`がcallableであれば組み込み関数ですので、`func`を`args`に適用して値を返します。

そうでなければユーザ定義関数の処理に入ります。

`[_, parameters, body] = func`は`["func", param, body]`から`param`と`body`を取り出しています。
言語によっては、`def _apply(self, [_, parameters, body], args)`のように書いていきなり取り出すこともできますがPythonではできません。残念。

`self._env = Environment(parent_env)`では、この関数を評価するためのローカルな環境を作ります[^dynamic]。
複文でスコープを導入したときと同じです。

[^dynamic]:「アレ？」と思われた方もいるかもしれませんがしばしお待ちを。

`for param, arg in zip(parameters, args): self._env.define(param, arg)`では、ふたつの配列からひとつずつ値を取っては処理を行うイディオムを用いて、`self._env.define()`を呼び出しています。
これにより、新たに作ったスコープにローカル変数を追加したことになります。
`func (a, b, c) { print a + b + c; }`という関数に`(5, 6, 7)`という実引数を適用したのであれば、`a`（値は5）、`b`（値は6）、`c`（値は7）というローカル変数が追加されます。

その状態で`self._eval_statement(body)`を実行すると、引数を指定した状態で`body`が実行される、というわけです。
かしこい！

`body`の評価を終えたら、今作ったスコープは不要になるので`self._env = parent_env`で捨ててしまいます。

ところで、まだreturnは実装していません。
returnなしで関数を評価し終わったら0を返すことにします。nullがあればnullを返すところですが。
returnはあとで実装します。

では実行です。緊張します（何
returnがないので結果はprintで見ます。

```
Input source and enter Ctrl+D:
var sum = func(a, b) {
    print a + b;
};
sum(5, 6); sum(7, 8);
Output:
<AST略>
11
15
```

できました！
返り値や即時実行、引数なしの場合やASTも見ておきます。

```
Input source and enter Ctrl+D:
print func() {}();
Output:
['program', ['print', [['func', [], ['block']]]]]
0
Input source and enter Ctrl+D:
func() { print 5; }();
Output:
['program', ['expr', [['func', [], ['block', ['print', 5]]]]]]
5
Input source and enter Ctrl+D:
func(a, b) { print a + b; }(5, 6);
Output:
['program', ['expr', [['func', ['a', 'b'], ['block', ['print', ['+', 'a', 'b']]]], 5, 6]]]
11
```

関数の評価中に環境がどうなってるか見ておきましょう。

```
Input source and enter Ctrl+D:
var sum = func(a, b) {
  var c = 4;
  print a + b + c;
  print_env();
};
sum(2, 3);
Output:
['program', ['var', 'sum', ['func', ['a', 'b'], ['block', ['var', 'c', 4], ['print', ['+', ['+', 'a', 'b'], 'c']], ['expr', ['print_env']]]]], ['expr', ['sum', 2, 3]]]
{'less': '<builtin>', 'print_env': '<builtin>', 'sum': '<func>'}
{'a': 2, 'b': 3}
{'c': 4}
9
```

`{'less': '<builtin>', 'print_env': '<builtin>', 'sum': '<func>'}`が最上位の環境で、組み込み関数と`sum`が定義されています。
`{'a': 2, 'b': 3}`が関数のための環境で、実引数の値が仮引数と紐づけられて保存されています。
`{'c': 4}`が最下層で、`body`を評価するときの環境です。関数内のローカル変数が保存されています。
`body`を評価しているときに`a`や`b`が出てきたら、最下層の環境から変数を探し始めますが、最下層にはないので上位の環境を探しに行き、そこで`a`や`b`を見つけます。

これが関数の評価のしくみでした。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8040_user_function_call) [差分](https://github.com/koba925/minilang-book/compare/8030_user_function_type...8040_user_function_call)

## 補足：「ふたつの配列からひとつずつ値を取っては処理を行うイディオム」について

お断り：
以下、「配列を返す」などと書いていますが雰囲気レベルで、正確な記述ではありません。
正確な動作は https://docs.python.org/3/library/functions.html#zip や https://docs.python.org/3/glossary.html#term-iterator をご覧ください。

> `for param, arg in zip(parameters, args): self._env.define(param, arg)`では、ふたつの配列からひとつずつ値を取っては処理を行うイディオムを用いています。

`func (a, b, c) { print a + b + c; }`という関数に`(5, 6, 7)`という実引数を適用したとして説明します。
このとき`parameters`は`["a", "b", "c"]`、`args`は`[5, 6, 7]`になっています。

`zip(parameters, args)`は以下のように`parameters`と`args`から値を一つずつとってきたタプルの配列を返します。

* `parameters`から`"a"`、`args`から`5`を取って`("a", 5)`を作る
* `parameters`から`"b"`、`args`から`6`を取って`("b", 6)`を作る
* `parameters`から`"b"`、`args`から`7`を取って`("c", 7)`を作る

これらを配列にして`[("a", 1), ("b", 2), ("c", 3)]`が値となります。

`for param, arg in zip(parameters, args): self._env.define(param, arg)`では、`[("a", 5), ("b", 6), ("c", 7)]`の先頭からひとつずつタプルをとり、`param`・`arg`の値として、`self._env.define()`を呼び出します。

* `("a", 5)` → `param = "a", arg = 5` → `self._env.define("a", 5)`を実行
* `("b", 6)` → `param = "b", arg = 6` → `self._env.define("b", 6)`を実行
* `("c", 7)` → `param = "c", arg = 7` → `self._env.define("c", 7)`を実行

全体として、「ふたつの配列からひとつずつ値を取っては処理」されました。
よくある処理なので、慣れておくと便利です。

## returnを実装する

return文を実装します。
returnの構文は`return;`または`return <式>;`、ASTは`["return", <式>]`とします。
`return;`と値が省略された場合は0を返します。nullがあれば（略

構文解析はもう問題ないでしょう。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
             case "while": return self._parse_while()
+            case "return": return self._parse_return()
             case "print": return self._parse_print()
             ...

     def _parse_while(self):
         ...
 
+    def _parse_return(self):
+        self._next_token()
+        value = 0
+        if self._current_token != ";": value = self._parse_expression()
+        self._consume_token(";")
+        return ["return", value]
+
     def _parse_print(self):
         ...
```

if文やwhile文の評価はそのままifやwhileで書けましたが、return文の評価は単純にreturnで、というわけにはいきません。
たとえば関数の中で`while ... { if ... { return; } }`のようなreturn文を評価する場合、`_eval_if`や`_eval_while`も抜けて`_apply`まで戻ってくる必要があります。
一気に呼び出しの階層を全部飛び越えるために、例外のしくみを使います。

まず、返り値を持ったExceptionのサブクラスを作ります。

```diff py
 class Parser:
     ...

+class Return(Exception):
+    def __init__(self, value): self.value = value

 class Environment:
     ...
```

return文を見つけたら、返り値の式を評価して`Return`例外を起こします。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
             case ["while", cond, body]: self._eval_while(cond, body)
+            case ["return", value]: raise Return(self._eval_expr(value))
             case ["print", expr]: self._eval_print(expr)
             ...
```

`_apply()`では、try～exceptの中で`body`を評価し、`Return`例外をつかまえて関数の返り値とします。

```diff py
     def _apply(self, func, args):
         if callable(func): return func(*args)
 
         [_, parameters, body] = func
         parent_env = self._env
         self._env = Environment(parent_env)
         for param, arg in zip(parameters, args): self._env.define(param, arg)
-        self._eval_statement(body)
-        self._env = parent_env
-        return 0
+        value = 0
+        try:
+            self._eval_statement(body)
+        except Return as ret:
+            value = ret.value
+        self._env = parent_env
+        return value
```

環境捨てるところすっとばして抜けちゃっていいの？と思われるかもしれませんが、見てる人がいなくなるのでいずれガベージコレクションで
回収されるはずです。たぶん。

```
print func() { return; }();"), [0])
print func() { return 5; }();"), [5])
print func(a, b) { return a + b; }(5, 6);"), [11])
func() { print 5; return; print 6; }();"), [5])

var sum = func(a, b) {
    return a + b;
};
print sum(5, 6);
print sum(7, 8);
```


minilangのreturnは数値だろうが真偽値だろうが関数だろうが分け隔てなく返します。
たとえばこんな風に。

```
Input source and enter Ctrl+D:
var add = func(a, b) { return a + b; };
var sub = func(a, b) { return a - b; };
var add_or_sub = func(op) { if op = 1 { return add; } else { return sub; } };
print  add_or_sub(2)(7, 3);
Output:
4
```

`add_or_sub(2)`が`sub`を返すので、`sub(7, 3)`を評価して`4`が帰ってくるというわけです。
ここでは定義済みのユーザ定義関数を返しましたが、`func`を直接返したり、組み込み関数を返したりも同様に可能です。

ループからもちゃんと抜けて帰ってきます。

```
Input source and enter Ctrl+D:
var nums_to_n = func(n) {
    var k = 1;
    while true {
        print k;
        if k = n { return; }
        set k = k + 1;
    }
};
nums_to_n(5);
Output:
[...]
1
2
3
4
5
```

gcdやfibももっと自然に書けるようになりました。

```
Input source and enter Ctrl+D:
var gcd = func(a, b) {
    var tmp = 0;
    while b # 0 {
        if less(a, b) {
            set tmp = a; set a = b; set b = tmp;
        }
        set a = a - b;
    }
    return a;
};
print gcd(36, 12);
Output:
[...]
12
```

```
Input source and enter Ctrl+D:
var fib = func(n) {
    if n = 1 { return 1; }
    if n = 2 { return 1; }
    return fib(n - 1) + fib(n - 2);
};
print fib(6);
Output:
[...]
8
```

直接自分自身に再帰するのではなく、ぐるっと回って自分自身を呼び出すような再帰も問題ありません。

```
Input source and enter Ctrl+D:
var is_even = func(a) { if a = 0 { return true; } else { return is_odd(a - 1); } };
var is_odd = func(a) { if a = 0 { return false; } else { return is_even(a - 1); } };
print is_even(5);
print is_odd(5);
print is_even(6);
print is_odd(6);
Output:
[...]
false
true
true
false
```

何をしているかわかるでしょうか？

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8050_return) [差分](https://github.com/koba925/minilang-book/compare/8040_user_function_call...8050_return)

## 動的スコープと静的スコープ・クロージャのお話

> 「アレ？」と思われた方もいるかもしれませんがしばしお待ちを。

注にこんなことを書きました。
いまどきのスクリプト言語は静的（static）スコープを実装しているものがほとんどなんですが、上で作った関数は動的（dynamic）スコープを実装しているのでした。

静的スコープはレキシカル（lexical）スコープとも呼ばれます。
（実行してみなくても）字面だけで決まるスコープ、といったところでしょうか。

というわけで、静的スコープとか動的スコープとかって何なの？という説明からしなくてはなりません。
静的（またはレキシカル）スコープはクロージャという概念とほぼセットで出てきます。
クロージャについても説明します。

知ってるよ！という人はこの節は飛ばしてかまいません。
また、以下の説明を読んでみてなんだかよくわからん！という方も、いったん気にせず次の節に進んでみてください。
実のところ、説明よりもコード見た方がわかりやすいのでは？という気もしています。
静的スコープ・クロージャがいったいなんなのか、動作原理から具体的に理解できるというのはインタプリタを勉強する効能の中でも大きいものではないかと思います。

では始めます。

`func(a) { return a + b; };`という関数は単独では評価できません。

```
Input source and enter Ctrl+D:
print func(a) { return a + b; } (5);
Output:
Error: `b` not defined.
```

特にC言語から来ている人（いるかな）にとっては当たり前の揺るがない事実のように思われるかもしれませんが、minilangのようなブロック構造を持つ言語ではそうでもありません。
`b`が外側で定義されていれば成功します。

```
Input source and enter Ctrl+D:
var b = 6;
print func(a) { return a + b; } (5);
Output:
[...]
11
```

これは、minilangは今見てる環境で見つからなかったら呼び出し元の環境を見に行くためです。
最上位の環境にあるbが見えています。

```
Input source and enter Ctrl+D:
var b = 6;
print func(a) { print_env(); return a + b; } (5);
Output:
{'less': '<builtin>', 'print_env': '<builtin>', 'b': 6}
{'a': 5}
{}
Output:
[...]
11
```

この`b`のように、関数の中で使われているけれども引数に含まれていない変数のことを自由変数と言います。


さらに、
（これは静的スコープだとエラー）

var fa = func(a) { print_env(); return a + b; };
var fb = func() {
  var b = 6;
  return fa(5);
};
print fb();

さらにさらに、こういうのだったら？

var fb = func() {
  var b = 6;
  return func(a) { return a + b; };
};
var fa = fb();
print fa(5);

今は前者の動きになっています。
関数の環境の中になければ呼び出し元の環境へ探しに行きます。

Input source and enter Ctrl+D:
var fa = func(a) { return a + b; };
print fa(5);
Output:
Error: `b` not defined.
Input source and enter Ctrl+D:
var b = 6;
print fa(5);
Output:
11

今のプログラミング言語では後者が主流
といっても前者が絶滅してるわけではなく、言語によっては指定すれば前者

静的スコープ・レキシカルスコープとも呼ばれます
定義時スコープとか生成時スコープとか言った方がわかりやすいのでは

envがくっついたことにより「閉じた」

むしろコードを見てもらった方がなんなのかよくわかるのでは

文法は何一つ変わらないのでEvaluatorの修正

var fb = func(b) {
  print_env();
  return func(a) { 
    print_env();
    return a + b;
  };
};

var fa = fb(6);
var b = 5;
print_env();
print fa(7);

var fb = func(b) {
  return func(a) { return a + b; };
};
var fa = fb(5);
print fa(6);

var b = 6;
print fa(7);

実は私、静的スコープやらレキシカルスコープとはこういうものです、と言われてもいまひとつピンと来てなかったんですよね。
実装後にもう一度触れます。

## 静的スコープ・クロージャの実装

説明が長くてどんな実装なんだろうと身構えてるかもしれませんが静的スコープ・クロージャの実装はあっけないほど簡単です。

構文は変わらないので`Parser`はそのままです。
では何が変わるのかというと「意味論」（semantics）が変わる、と言ったりします。
その構文は実際どういうことなんだ、というところが変わるということですね。

まず`func`で関数を「作るとき」に、そのときの環境を覚えておきます。
関数のASTに`self._env`を追加しているところがそれです。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
             case str(name): return self._eval_variable(name)
-            case ["func", param, body]: return ["func", param, body]
+            case ["func", param, body]: return ["func", param, body, self._env]
             case ["^", a, b]: return self._eval_expr(a) ** self._eval_expr(b)
             ...
```

そして、関数を適用するときには現在の環境（`self._env`）ではなく、関数のASTに覚えておいた環境を使います。

```diff py
     def _apply(self, func, args):
         if callable(func): return func(*args)
 
-        [_, parameters, body] = func
+        [_, parameters, body, env] = func
         parent_env = self._env
-        self._env = Environment(parent_env)
+        self._env = Environment(env)
         for param, arg in zip(parameters, args): self._env.define(param, arg)
         value = 0
         ...
```

これだけです。
動かしてみましょう。

```
Input source and enter Ctrl+D:
var fb = func() {
    var b = 6;
    return func(a) { return a + b; };
};
var fa = fb();
print fa(5);
Output:
[...]
11
```

`func(a)`を定義したときに`b`がスコープに存在しているので、`fb`を抜けた後に`fa`を評価するときも`b`が存在していて、5+6で11が返ります。

引数も同様に覚えていてくれます。

```
Input source and enter Ctrl+D:
var make_adder = func(a) {
    return func(b) { return a + b; };
};
var a = 5;
var add_6 = make_adder(6);
print add_6(7);
Output:
[...]
13
```

`make_adder(6)`で、引数の`a`に6が入った状態で`func`が作成されます。
その環境を記憶しているため、`add_6(7)`を評価するときも`a`は6なので、13が返ってきます。
`var a = 5;`と`a`を定義していますが、こちらの`a`は`add_6(7)`の評価には影響を与えていないことがわかります。

関数を呼び出したときの環境で自由変数を探すのが動的スコープ、関数を作ったときの環境で自由変数を探すのが静的スコープ、ということがピンときたでしょうか？
自分はインタプリタを勉強してみてやっとそのあたりが腑に落ちました。
静的スコープ・動的スコープじゃなくて、定義時スコープ・評価時スコープとか言ってくれてたらピンと来たんじゃないかという気がします。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8060_closure) [差分](https://github.com/koba925/minilang-book/compare/8050_return...8060_closure)

ここからちょっとだけ余談です。
このコードは成功するでしょうか失敗するでしょうか？

```
var fa = func(a) { return a + b; };
var b = 6;
print fa(5);
```

動的スコープであれば、`fa`の呼び出し時に`b`が定義されているので成功しそうですし実際成功します。
静的スコープだと、`fa`の定義時には`b`が定義されていないので失敗しそうですが、実は成功します。
環境を表示させながら実行してみましょう。

```
Input source and enter Ctrl+D:
var fa = func(a) { print_env(); return a + b; };
var b = 6;
print fa(5);
Output:
[...]
{'less': '<builtin>', 'print_env': '<builtin>', 'fa': '<func>', 'b': 6}
{'a': 5}
{}
11
```

これは、`fa`を定義したときに環境を覚えておくのですが、`b`は環境の最上位に追加されるので、`fa`を定義したときの環境を遡っていくと最上位で`b`が見つかる（見つかってしまう）ためです。
厳密な意味での静的スコープではエラーになるのが正しいのかもしれませんが、Pythonでも普通に実行できるので気にしないことにします。

```py
$ python
>>> def f(a): return a + b
... 
>>> b = 6
>>> print(f(5))
11
```

## 関数を定義する文を作る

最後、おまけのようなものですが`var add = func(a, b) { ... }`などと書く代わりに`def add(a, b) { ... }`と書けるようにします。
文の先頭で`func`を見つけたら`var`の形に直してASTを作ればOKです。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
             case "while": return self._parse_while()
+            case "def": return self._parse_def()
             case "return": return self._parse_return()
             ...
     
     ...

     def _parse_while(self):
         ...
 
+    def _parse_def(self):
+        self._next_token()
+        name = self._parse_primary()
+        assert isinstance(name, str),  f"Expected a name, found `{name}`."
+        params = self._parse_parameters()
+        body = self._parse_block()
+        return ["var", name, ["func", params, body]]

     def _parse_return(self):
         ...
```

elifでも似たようなことをしましたね。


この`def`のように、より基礎的な要素を使って書くこともできるけれども人間が読みやすくするために追加されたものを構文糖衣（シンタックスシュガー）と呼びます[^elif-sugar]。
言語処理系の中でもより基礎的な要素に還元して処理することができます。

[^elif-sugar]: ただ、elifがシンタックスシュガーと呼ばれていることはあまりない気がします。
基礎部品だと思われてるからでしょうか。

Evaluatorは変更不要です。

```
Input source and enter Ctrl+D:
def sum(a, b) {
    return a + b;
}
print sum(2, 3);
print sum(4, 5);
Output:
['program', ['var', 'sum', ['func', ['a', 'b'], ['block', ['return', ['+', 'a', 'b']]]]], ['print', ['sum', 2, 3]], ['print', ['sum', 4, 5]]]
5
9
Input source and enter Ctrl+D:
def fib(n) {
    if n = 1 { return 1; }
    if n = 2 { return 1; }
    return fib(n - 1) + fib(n - 2);
}
print fib(6);
Output:
[...]
8
Input source and enter Ctrl+D:
def gcd(a, b) {
    var tmp = 0;
    while b # 0 {
        if less(a, b) {
            set tmp = a; set a = b; set b = tmp;
        }
        set a = a - b;
    }
    return a;
}
print gcd(36, 12);
Output:
[...]
12
```

この本でのminilang実装はここまでです。
おつかれさまでした！

GitHub: [コード](https://github.com/koba925/minilang-book/tree/8070_def) [差分Z](https://github.com/koba925/minilang-book/compare/8060_closure...8070_def)