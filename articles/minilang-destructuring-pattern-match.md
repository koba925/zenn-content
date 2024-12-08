---
title: "minilangに分割代入やパターンマッチを実装する"
emoji: "➗"
type: "tech"
topics: ["computerscience", "language", "interpreter", "minilang"]
published: true
---

※「[350行くらいのPythonで作るプログラミング言語実装超入門](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines)」の関連記事で[70行のminilangでつくる評価器超入門（うそ](https://zenn.dev/kb84tkhr/articles/miniminieval)の続きです。これまでの記事は[minilangの記事一覧 | Zenn](https://zenn.dev/topics/minilang)をご覧ください。

前回の`eval()`を実装しているとこのへんとか

```
        var params = op[0]; var body = op[1]; var env = op[2];
```

このへんをなんとかしたくなりました。

```
        if expr[0] = 'var' { this._env.define(expr[1], this.eval(expr[2])); return; }
        if expr[0] = 'set' { this._env.assign(expr[1], this.eval(expr[2])); return; }
        ...
        if expr[0] = 'func' { return [first(rest(expr)), rest(rest(expr)), this._env]; }
```

ということで分割代入とパターンマッチを作ってみます。いろいろやろうとすると大変そうなので基本的なものだけ。

## 分割代入を実装する

```
        var params = op[0]; var body = op[1]; var env = op[2];
```

を

```
        var [params, body, env] = op;
```

と書けるようにします。もうすこし詳しく言うとこんな感じ。

* `set`でも使える
* 左辺は入れ子になっていても大丈夫 `var [a, [b, c]] = [1, [2, 3]];`
* 使わない値は`_`でもOK（形は合っている必要がある） `var [a, _, _] = [1, 2, 3];`
* `-`をつけると残り全部 `var [first, -rest] = [1, 2, 3];`
* swapもできる `set [a, b] = [b, a];`
* 辞書には対応していない

コードの説明です。

`var`や`set`の左辺は自前でparseしていましたが、入れ子も出てきたので`_parse_call_index_dot()`に任せます。`var`や`set`が対応していない形も通してしまいますが、評価のときにチェックします。

```diff py
     def _parse_var_set(self):
         op = self._current_token
         self._next_token()
-        target = self._parse_primary()
-        assert isinstance(target, str),  f"Expected a name, found `{target}`."
-        while (left := self._current_token) in ("[", "."):
-            self._next_token()
-            if left == "[":
-                index = self._parse_expression()
-                target = ["index", target, index]
-                self._consume_token("]")
-            else:
-                index = self._current_token
-                target = ["dot", target, ["str", str(index)]]
-                self._next_token()
+        target = self._parse_call_index_dot()
         self._consume_token("=")
         value = self._parse_expression()
         self._consume_token(";")
         return [op, target, value]
```

事情がありまして右辺は先に評価を済ませておきます。

```diff py
     def _eval_statement(self, statement):
         match statement:
             case ["block", *statements]: self._eval_block(statements)
-            case ["var", name, value]: self._eval_var(name, value)
+            case ["var", name, value]: self._eval_var(name, self._eval_expr(value))
-            case ["set", name, value]: self._eval_set(name, value)
+            case ["set", name, value]: self._eval_set(name, self._eval_expr(value))
             case ["if", cond, conseq, alt]: self._eval_if(cond, conseq, alt)
             ...
```

`var`の評価です。ここが山場です。`var`の左辺（`target`）の形によって処理を分けます、

```diff py
     def _eval_var(self, target, value):
-        self._env.define(target, self._eval_expr(value))
+        match target:
+            case ["arr", [["-", target]]]:
+                match value:
+                    case ["arr", values]:
+                        self._env.define(target, value)
+                        return
+            case ["arr", targets]:
+                match value:
+                    case ["arr", values]:
+                        if not targets and not values: return
+                        if not targets: assert False, f"Too many values."
+                        target, *rest_targets = targets
+                        if not values: assert False, f"Too many targets."
+                        value, *rest_values = values
+                        if target != "_": self._eval_var(target, value)
+                        self._eval_var(["arr", rest_targets], ["arr", rest_values])
+                        return
+            case str(name):
+                self._env.define(target, value)
+                return
+        assert False, f"Illegal declaration."
```

さらにパーツごとに分けます。

左辺が文字列だったら変数と言うことなので単純に変数を定義します。ここでは再帰呼び出しは行わず、基本の基底ケースとなります。

```diff py
+            case str(name):
+                self._env.define(target, value)
+                return
```

次が本当の山場で、左辺が配列の場合です。配列の入れ子にも対応するので再帰的に処理します。

1. 右辺も配列であることを確認
2. 左辺・右辺とも空であればなにもせず終了
3. 左辺・右辺どちらかが残っていたらエラー
4. 先頭の要素を取り出し、`_eval_var()`を再帰的に呼び出す
   ただし左辺が`_`の場合は無視
5. 残りの要素についても`_eval_var()`を再帰的に呼び出す

変数の宣言は基底ケースに任せてここでは処理の流れをコントロールするだけです。

```diff py
+            case ["arr", targets]:
+                match value:
+                    case ["arr", values]:
+                        if not targets and not values: return
+                        if not targets: assert False, f"Too many values."
+                        target, *rest_targets = targets
+                        if not values: assert False, f"Too many targets."
+                        value, *rest_values = values
+                        if target != "_": self._eval_var(target, value)
+                        self._eval_var(["arr", rest_targets], ["arr", rest_values])
+                        return
```

木構造の処理では典型的なパターンですが、再帰に慣れていないとなんでこれでうまくいっちゃうの？ってなるかもしれません。[再帰の読み方](https://zenn.dev/kb84tkhr/articles/how-to-read-recursion)を参考に「再帰する呼び出しは正しいと信じて、それ以上掘り下げない」読み方をしてみてください。`self._eval_var(target, value)`がちゃんと先頭の要素を処理して、`self._eval_var(["arr", rest_targets], ["arr", rest_values])`がちゃんと残りの要素を処理してくれれば全部うまくいくはずですね？

さきほどの「事情がありまして」というのは、`_eval_var()`の中で`_eval_expr()`を呼んでしまうと再帰のたびに評価を行ってしまうからです。再帰用の関数をさらに分ける方法もありますがここでは簡単に済ませました。

以下のcaseは、配列の形の最後に`-<名前>`という形が来た時の処理です。右辺も配列であれば、配列の残りをまるごと変数の値として定義します。こちらも再帰呼び出しをしないので基底ケースのひとつです。`case ["arr", targets]:`の形よりも先に判定する必要があるので先頭に持ってきています。

```diff py
+            case ["arr", [["-", target]]]:
+                match value:
+                    case ["arr", values]:
+                        self._env.define(target, value)
+                        return
```

どの形にもマッチしなければエラーです。

```diff py
+        assert False, f"Illegal declaration."
```

setも同様です。`_env.define()`の代わりに`_env.assign()`を呼び出しているくらい。`"index"`や`"dot"`の処理が入っているのは元からです。共通部分を関数にくくりだすか・・・？と思いましたが読みにくくなりそうでやめておきました。

```diff py
     def _eval_set(self, target, value):
         match target:
+            case ["arr", [["-", target]]]:
+                match value:
+                    case ["arr", values]:
+                        self._env.assign(target, value)
+                        return
+            case ["arr", targets]:
+                match value:
+                    case ["arr", values]:
+                        if not targets and not values: return
+                        if not targets: assert False, f"Too many values."
+                        target, *rest_targets = targets
+                        if not values: assert False, f"Too many targets."
+                        value, *rest_values = values
+                        if target != "_": self._eval_set(target, value)
+                        self._eval_set(["arr", rest_targets], ["arr", rest_values])
+                        return
             case ["index", expr, index] | ["dot", expr, index]:
                 match [self._eval_expr(expr), self._eval_expr(index)]:
                     case [["arr", values], int(ind)] | [["dic", values], str(ind)]:
-                        values[ind] = self._eval_expr(value)
+                        values[ind] = value
                         return
             case str(name):
-                self._env.assign(target, self._eval_expr(value))
+                self._env.assign(target, value)
                 return
         assert False, f"Illegal assignment."
```

`value`が評価済みで渡されるので、`set [a, b] = [b, a];`で変数の値を入れ替えることができます。分割代入の利用パターンのひとつ。

varからやってみます。ASTの出力は削除で。

```
Input source and enter Ctrl+D:
var [a, [b, c]] = [5, [6, 7]]; print a; print b; print c;
Output:
5
6
7
Input source and enter Ctrl+D:
var [e, f] = [[5, 6], 7]; print e; print f;
Output:
[5, 6]
7
Input source and enter Ctrl+D:
var [_, g, _] = [5, 6, 7]; print g;
Output:
6
Input source and enter Ctrl+D:
var [h, -i] = [5, 6, 7]; print h; print i;
Output:
5
[6, 7]
Input source and enter Ctrl+D:
var [j, k] = [5, 6, 7];
Output:
Error: Too many values.
Input source and enter Ctrl+D:
var [3, l] = [5, 6];
Output:
Error: Illegal declaration.
```

setをやります。swapも。

```
Input source and enter Ctrl+D:
var [a, b, c] = [2, 3, 4]; set [a, [b, c]] = [5, [6, 7]]; print a; print b; print c;
Output:
5
6
7
Input source and enter Ctrl+D:
var [e, f] = [2, 3]; set [e, f] = [[5, 6], 7]; print e; print f;
Output:
[5, 6]
7
Input source and enter Ctrl+D:
var g = 2; set [_, g, _] = [5, 6, 7]; print g;
Output:
6
Input source and enter Ctrl+D:
var [h, i] = [2, 3]; set [h, -i] = [5, 6, 7]; print h; print i;
Output:
5
[6, 7]
Input source and enter Ctrl+D:
var [j, k] = [2, 3]; set [j, k] = [5, 6, 7];
Output:
Error: Too many values.
Input source and enter Ctrl+D:
var l = 2; set [3, l] = [5, 6];
Output:
Error: Illegal assignment.
Input source and enter Ctrl+D:
var [m, n] = [2, 3]; set [m, n] = [n, m]; print m; print n;
Output:
3
2
```

ところで、minilangでは宣言と代入が分かれています。そのため分割したあとに未宣言の変数と宣言済みの変数に同時に値を割り当てることができません。

```
Input source and enter Ctrl+D:
var a = 5;
Output:

Input source and enter Ctrl+D:
var [a, b] = [6, 7];
Output:
Error: `a` already defined.
Input source and enter Ctrl+D:
set [a, b] = [6, 7];
Output:
Error: `b` not defined.
```

pythonだとできます。

```py
>>> a = 5
>>> [a, b] = [6, 7]
>>> [a, b]
[6, 7]
```

そんなに不便ではない気がしますがどうでしょうね？

GitHub: [コード](https://github.com/koba925/minilang-book/blob/destructuring/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/update_readme...destructuring/)

## forでも分割代入

分割代入ができるなら`for [a, b] in [[1, 2], [3, 4], ...]`のようにも書けるようにしたい！

`for <変数> in ...`だったのを`for <式> in ...`でいいことにします。

```diff py
     def _parse_for(self):
         self._next_token()
-        var = self._parse_primary()
-        assert isinstance(var, str),  f"Expected a name, found `{var}`."
+        target = self._parse_expression()
         self._consume_token("in")
         col = self._parse_expression()
         self._check_token("{")
         body = self._parse_block()
-        return ["for", var, col, body]
+        return ["for", target, col, body]
```

評価時は、いったん宣言しておいてから、ループするごとに値を代入するようにします。ループごとに代入する方は`_eval_set()`がそのまま使えましたが宣言は`_eval_var()`をキレイに流用する方法が思いつかなくてあらためて書いてます。`_eval_var()`との違いは、宣言するだけで値が不要というだけで、うまくやれば流用できそうな気がするんですけれども・・・

```diff py  
-    def _eval_for(self, var, col, body):
+    def _eval_for(self, target, col, body):
+        def define_vars(target):
+            match target:
+                case ["arr", [["-", target]]]:
+                    self._env.define(target, None)
+                    return
+                case ["arr", targets]:
+                    if not targets: return
+                    target, *rest_targets = targets
+                    if target != "_": define_vars(target)
+                    define_vars(["arr", rest_targets])
+                    return
+                case str(name):
+                    self._env.define(name, None)
+                    return
+            assert False, f"Illegal for loop target `{target}`."
+
         parent_env = self._env
         self._env = Environment(parent_env)
-        self._env.define(var, None)
+        define_vars(target)
         match self._eval_expr(col):
             case ["arr", value] | str(value): col = value
             case ["dic", value]: col = value.keys()
-        for val in col:
-            self._env.assign(var, val)
+        for value in col:
+            self._eval_set(target, value)
             try: self._eval_statement(body)
             except Continue: continue
             except Break: break
         self._env = parent_env
```

動かします。

```
Input source and enter Ctrl+D:
for [i, b] in [[1, true], [2, false]] { print [i, b]; }
Output:
[1, true]
[2, false]
Input source and enter Ctrl+D:
for [i, -bs] in [[1], [2, true], [3, true, false]] { print [i, bs]; }
Output:
[1, []]
[2, [true]]
[3, [true, false]]
Input source and enter Ctrl+D:
for [_, b] in [[1, true], [2, false]] { print b; }
Output:
true
false
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/destructuring_for/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/destructuring...destructuring_for)

## パターンマッチを実装する

似たようなやりかたでmatchも実装できるはず・・・！構文は余計なもの入れずにこんな感じで。

```
match [1, [2, 3]] {
  [a, b, c] { ... }
  [[a, b], c] { ... }
  [a, [b, c]] { ... }
}
```

できることは`var`や`set`と似てますが`or`で複数のパターンを並べることができるようにします。また、パターン中に定数が現れたら、左辺と右辺で同じ値かどうかもチェックします。

構文解析。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
+            case "match": return self._parse_match()
             ...
     ...

+    def _parse_match(self):
+        self._next_token()
+        expr = self._parse_expression()
+        self._consume_token("{")
+        cases = []
+        while self._current_token != "}":
+            cases.append(self._parse_case())
+        self._next_token()
+        return ["match", expr, cases]
+
+    def _parse_case(self):
+        pattern = self._parse_expression()
+        self._check_token("{")
+        block = self._parse_block()
+        return [pattern, block]
     ...
```

評価。ケースごとに環境を作成してはvar同様の形で再帰しながらパターンを走査し、変数があれば都度宣言していきます。今回は内部関数`match()`に分けて呼び出す形にしました。`match()`は全体がマッチすれば`True`を、そうでなければ`False`を返します。マッチしてたらループを抜けます。マッチしてもマッチしなくても環境は捨てる。環境を捨ててしまうので、マッチさせた変数が有効なのはmatchの中だけです。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
+            case ["match", expr, cases]: self._eval_match(expr, cases)
             ...
     ...
 
+    def _eval_match(self, expr, cases):
+        def match(pattern, value):
+            match pattern:
+                case ["or", left, right]:
+                    return match(left, value) or match(right, value)
+                case ["arr", [["-", name]]]:
+                    match value:
+                        case ["arr", _]:
+                            self._env.define(name, value)
+                            return True
+                case ["arr", pats]:
+                    match value:
+                        case ["arr", vals]:
+                            if not pats and not vals: return True
+                            if not pats or not vals: return False
+                            pat, *rest_pats = pats
+                            val, *rest_vals = vals
+                            return ((pat == "_" or match(pat, val)) and
+                                    match(["arr", rest_pats], ["arr", rest_vals]))
+                case int(pat) | bool(pat) | ["str", pat]:
+                    return pat == value
+                case str(name):
+                    if name != "_": self._env.define(name, value)
+                    return True
+            return False
+
+        value = self._eval_expr(expr)
+        for pattern, block in cases:
+            parent_env = self._env
+            self._env = Environment(parent_env)
+            try:
+                if match(pattern, value):
+                    self._eval_statement(block)
+                    return
+            finally:
+                self._env = parent_env
     ...
```

今までと類似の処理を行っているところは説明を割愛させていただいて。

以下の部分は`or`に対応していて、どちらかがマッチすればTrueを返します。

```diff py
+                case ["or", left, right]:
+                    return match(left, value) or match(right, value)
```

以下の部分はパターンに定数が現れた場合です。右辺と同じ値ならTrueを、そうでなければFalseを返します。

```diff py
+                case int(pat) | bool(pat) | ["str", pat]:
+                    return pat == value
```

ではやってみます。

```
Input source and enter Ctrl+D:
match 5 { 3 { print 4; } 5 { print 6; } }
Output:
6
Input source and enter Ctrl+D:
match 6 { 3 { print 4; } 5 { print 6; } }
Output:

Input source and enter Ctrl+D:
match [1, [2, 3]] { [a, b, c] {} [[a, b], c] {} [a, [b, c]] { print [a, b, c]; } }
Output:
[1, 2, 3]
Input source and enter Ctrl+D:
match [1, 2, 3] { 4 { print 5; } [a, -b] { print b; } } 
Output:
[2, 3]
Input source and enter Ctrl+D:
match 4 { 3 or 4 { print 5; } }
Output:
5
Input source and enter Ctrl+D:
match [1, 2, 3] { 4 { print 5; } [1, _, _] { print 6; } }
Output:
6
```

できました。

そういえば、パターンマッチというとこういうことできないといけなかった気がします。今の実装だとエラーです。

```
Input source and enter Ctrl+D:
match [3, 3] { [a, a] { print 'same elements'; } }
Output:
Error: `a` already defined.
```

`a`に3が入れば`[3, 3]` = `[a, a]`でつじつまがあうのでマッチしてほしいところですが・・・

```py
>>> match [3, 3]:
...     case [a, a]: print(1)
... 
SyntaxError: multiple assignments to name 'a' in pattern
```

Pythonでもエラーでした。よし大丈夫（何が

GitHub: [コード](https://github.com/koba925/minilang-book/blob/match/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/destructuring_for...match)

## evalを書き直す

道具ができたのでminilang版評価器を更新しました。

`eval`はmatchのかたまりに。直で`[0]`とか`[2]`とか書かなくてよくなったのがうれしい。

```diff
 var Evaluator = $[
     eval: func(this, expr) {
-        if type(expr) = 'int' or type(expr) = 'bool' { return expr; }
-        if type(expr) = 'str' { return this._env.get(expr); }
-        if expr[0] = 'var' { this._env.define(expr[1], this.eval(expr[2])); return; }
-        if expr[0] = 'set' { this._env.assign(expr[1], this.eval(expr[2])); return; }
-        if expr[0] = 'if' {
-            if this.eval(expr[1]) {
-                return this.eval(expr[2]);
-            } else {
-                return this.eval(expr[3]);
-        if expr[0] = 'func' { return [first(rest(expr)), rest(rest(expr)), this._env]; }
-        return this._apply(expr);
+        match type(expr) {
+            'int' or 'bool' { return expr; }
+            'str' { return this._env.get(expr); }
+        }
+        match expr {
+            ['var', name, val] { this._env.define(name, this.eval(val)); return; }
+            ['set', name, val] { this._env.assign(name, this.eval(val)); return; }
+            ['if', cond, conseq, alt] {
+                if this.eval(cond) {
+                    return this.eval(conseq);
+                } else {
+                    return this.eval(alt);
+                }
             }
+            ['func', params, -body] { return [params, body, this._env]; }
+            _ { return this._apply(expr); }
+        }
     },
```

`_apply`では分割代入を駆使します。`var [op, -args] = expr.map(this.eval);`とか`for [param, arg] in zip(params, args)`って書けるようになったのはとてもうれしい。`map()`とか`zip()`は別途用意してやる必要がありますけど。

```diff
 _apply: func(this, expr) {
-    var op = this.eval(first(expr));
-    var args = []; for a in rest(expr) { push(args, this.eval(a)); }
+    var [op, -args] = expr.map(this.eval);
     if type(op) = 'func' { return op(args); }
-    var params = op[0]; var body = op[1]; var env = op[2];
+    var [params, body, env] = op;
     var cur_env = this._env; set this._env = environment(env);
-    this._add_args(params, args);
-    var ret = null; for expr in body { set ret = this.eval(expr); }
+    for [param, arg] in zip(params, args) { this._env.define(param, arg); }
+    var ret = null; body.map(func(expr) { set ret = this.eval(expr); });
     set this._env = cur_env;
     return ret;
 },
-_add_args: func(this, params, args) {
-    if params = [] { return; }
-    this._env.define(first(params), first(args));
-    this._add_args(rest(params), rest(args));
-}
```

`map()`とか`zip()`とか。標準ライブラリがほしい？

```diff
+def reduce(array, f, init) {
+    var result = init;
+    for e in array { set result = f(result, e); }
+    return result;
+}
+def map(array, f) {
+    return array.reduce(func(acc, e) { acc.push(f(e)); return acc; }, []);
+}
+def zip(array1, array2) {
+    if array1 = [] or array2 = [] { return []; }
+    return [[first(array1), first(array2)]] + zip(rest(array1), rest(array2));
+}
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/match/test_minilang.py) （下の方の`test_eval()`）
