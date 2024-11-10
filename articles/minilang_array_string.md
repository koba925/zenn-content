---
title: "minilangに配列と文字列を追加する"
emoji: "🔤"
type: "tech"
topics: ["computerscience", "language", "interpreter", "python", "minilang"]
published: true
---

※「[350行くらいのPythonで作るプログラミング言語実装超入門](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines)」の関連記事です。

minilangで配列と文字列を扱えるようにします。[minilangの宿題をやる](https://zenn.dev/articles/minilang-homework/)の続編です。

今回は少しだけ説明を入れます。

## 配列型を追加する

いつもどおり、まずは型だけ導入します。

minilangでは整数はPythonの整数を、真偽値はPythonの真偽値をそのまま使ってASTや値にしてきましたので、配列もPythonの配列を、と行きたいところなんですがASTを配列で表してたり値でも関数を表すのに`["func", ...]`と配列を使ったりしているので、配列をただの配列にしてしまっては見分けがつかなくなってしまいます。そこで、`[1, 2, 3]`という配列はASTや値では`["arr", [1, 2, 3]]`と表すことにします。

構文解析はいつもどおりです。

```diff py
     def _parse_primary(self):
         match self._current_token:
             ...
+            case "[": return self._parse_array()
             ...
 
+    def _parse_array(self):
+        self._next_token()
+        array = []
+        while self._current_token != "]":
+            array.append(self._parse_expression())
+            if self._current_token != "]":
+                self._consume_token(",")
+        self._consume_token("]")
+        return ["arr", array]
```

評価のときも、各要素を評価したら`["arr", ...]`で囲ってやります。

```diff py 
     def _to_print(self, value):
         match value:
             ...
+            case ["arr", values]:
+                return "[" + ", ".join([self._to_print(value) for value in values]) + "]"
             ...
 
     def _eval_expr(self, expr):
         match expr:
             ...
+            case ["arr", exprs]:
+                return ["arr", [self._eval_expr(expr) for expr in exprs]]
             ...
```

値と表示を作っただけですがPythonさんのユルさのおかげで比較なんかは既存のコードで動いてしまいます。

```
Input source and enter Ctrl+D:
print [1, true, false, less, func () {}, null, []];
Output:
['program', ['print', ['arr', [1, True, False, 'less', ['func', [], ['block']], None, ['arr', []]]]]]
[1, true, false, <builtin>, <func>, null, []]
Input source and enter Ctrl+D:
print [5, 6] = [5, 6];
print [5, 6] # [5, 6];
print [5, 6] < [5, 7];
print [5, 6] > [5, 7];
Output:
[...]
true
false
true
false
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/add_array/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/print_int_as_str...add_array)

※配列を追加する前に、整数をprintするとき文字列としてprintするよう修正しています

## 配列の+と*に対応する

配列の比較はなにもしなくてもできてしまったので、もしかして`+`や`*`もいけてしまうのでは？

```
Input source and enter Ctrl+D:
print [5, 6] + [7, 8];
Output:
['program', ['print', ['+', ['arr', [5, 6]], ['arr', [7, 8]]]]]
Error: `['arr', [5, 6], 'arr', [7, 8]]` unexpected in `_to_print`.
Input source and enter Ctrl+D:
print [5, 6] * 3;
Output:
['program', ['print', ['*', ['arr', [5, 6]], 3]]]
Error: `['arr', [5, 6], 'arr', [5, 6], 'arr', [5, 6]]` unexpected in `_to_print`.
```

あーそうですよねだめですよね

PythonはけなげにPythonなりの`+`や`*`をやってくれてますが、`["arr", ...]`が余分です。配列の時は特別扱いしてあげましょう。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
-            case ["*", a, b]: return self._eval_expr(a) * self._eval_expr(b)
+            case ["*", a, b]: return self._eval_mul(self._eval_expr(a), self._eval_expr(b))
             ...
-            case ["+", a, b]: return self._eval_expr(a) + self._eval_expr(b)
+            case ["+", a, b]: return self._eval_plus(self._eval_expr(a), self._eval_expr(b))
             ...
 
+    def _eval_mul(self, a, b):
+        match [a, b]:
+            case [["arr", values], int(times)]:
+                return ["arr", values * times]
+        return a * b
+
+    def _eval_plus(self, a, b):
+        match [a, b]:
+            case [["arr", values_a], ["arr", values_b]]:
+                return ["arr", values_a + values_b]
+        return a + b
```

配列と整数を`*`したり配列と配列を`+`しようとしたときはパターンマッチで中身を取り出してから`*`や`+`を行って、あらためて`["arr", ...]`で囲ってやります。

```
Input source and enter Ctrl+D:
print [5, 6] + [7, 8];
Output:
['program', ['print', ['+', ['arr', [5, 6]], ['arr', [7, 8]]]]]
[5, 6, 7, 8]
Input source and enter Ctrl+D:
print [5, 6] * 3;
Output:
['program', ['print', ['*', ['arr', [5, 6]], 3]]]
[5, 6, 5, 6, 5, 6]
```

できました。

GitHub: [コード](https://github.com/koba925/minilang-book/blob/arr_plus_mul/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/add_array...arr_plus_mul)

## 配列要素の参照を実装する

配列の要素を`[]`で参照できるようにします。`[]`は関数呼び出しの`()`と同じ優先度に置きます。

構文解析では`_parse_call()`を書き換えて`()`も`[]`も扱えるようにします。`left`に今どちらを処理しているのか覚えておき、それにしたがって処理を行います。`[]`のASTは`["index", <配列>, <インデックス>]`の形とします。

```diff py
     def _parse_power(self):
-        power = self._parse_call()
+        power = self._parse_call_array()
         ... 

-    def _parse_call(self):
+    def _parse_call_array(self):
+        def terminator(left):
+            return {"(": ")", "[": "]"}[left]
+
-        call = self._parse_primary()
+        result = self._parse_primary()
-        while self._current_token == "(":
+        while (left := self._current_token) in ("(", "["):
             self._next_token()
             args = []
-            while self._current_token != ")":
+            while self._current_token != terminator(left):
                 args.append(self._parse_expression())
-                if self._current_token != ")":
+                if self._current_token != terminator(left):
                     self._consume_token(",")
-            call = [call] + args
+            if left == "(": result = [result] + args
+            else: result = ["index", result, args[0]]
-            self._consume_token(")")
+            self._consume_token(terminator(left))
-        return call
+        return result
```

差分で示すとちょっとごちゃっとしてますね。見づらいようでしたら下にコードへのリンクがありますのでそちらも合わせてご参照ください。

評価では、`["index", <配列>, <インデックス>]`の形が来たら配列の実体を取り出してインデックスの指す要素を返します。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
+            case ["index", expr, index]:
+                return self._eval_index(self._eval_expr(expr), self._eval_expr(index))
             ...

+    def _eval_index(self, array, index):
+        match array:
+            case ["arr", value]:
+                return value[index]
+            case _:
+                assert False, "Index must be applied to an array."
```

変数に入れても（ちょっと見慣れない形ですが）入れなくても扱えます。

```
Input source and enter Ctrl+D:
var a = [5, 6, 7];
print a[1];
Output:
['program', ['var', 'a', ['arr', [5, 6, 7]]], ['print', ['index', 'a', 1]]]
6
Input source and enter Ctrl+D:
print [5, 6, 7][2];
Output:
['program', ['print', ['index', ['arr', [5, 6, 7]], 2]]]
7
```

２次元配列も扱えます。たぶん多次元でも。

```
Input source and enter Ctrl+D:
print [[5, 6], [7, 8]][1];
Output:
['program', ['print', ['index', ['arr', [['arr', [5, 6]], ['arr', [7, 8]]]], 1]]]
[7, 8]
Input source and enter Ctrl+D:
print [[5, 6], [7, 8]][1][0];
Output:
['program', ['print', ['index', ['index', ['arr', [['arr', [5, 6]], ['arr', [7, 8]]]], 1], 0]]]
7
```

`()`と`[]`が並んでいてもちゃんと動きます。

```
Input source and enter Ctrl+D:
print func(){ return [5, 6, 7, 8]; }()[0];
Output:
[...]
5
Input source and enter Ctrl+D:
print [func(i){ return [5, 6, 7, 8][i]; }][0](3);
Output:
[...]
8
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/arr_reference/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/arr_plus_mul...arr_reference)

## 配列要素の更新を実装する

配列の要素への代入に対応します。

今までsetの左辺は識別子が来ることになってましたが`<識別子>[<式>]...`という形も認めることにします。ASTは参照と同じ形です。varの左辺に配列の要素が来ても意味がないのでエラーにすべきところですが気が付かなかったことにします（いつもどおり

```diff py
     def _parse_var_set(self):
         op = self._current_token
         self._next_token()
-        name = self._parse_primary()
-        assert isinstance(name, str),  f"Expected a name, found `{name}`."
+        target = self._parse_primary()
+        assert isinstance(target, str),  f"Expected a name, found `{target}`."
+        while self._current_token == "[":
+            self._next_token()
+            index = self._parse_expression()
+            target = ["index", target, index]
+            self._consume_token("]")
         self._consume_token("=")
         value = self._parse_expression()
         self._consume_token(";")
-        return [op, name, value]
+        return [op, target, value]
```

`_parse_call_array()`にも修正が入ってますがこれは関数にしなくてもただの辞書でいいじゃないと気付いたので直したものです。動きは変わっていません。別のコミットにすればよかったんですけど。

評価では左辺が`["index", expr, index]`の形だったら`expr`の`index`番目に代入します。識別子だったら従来通り。

```diff py
-    def _eval_var(self, name, value):
+    def _eval_var(self, target, value):
-        self._env.define(name, self._eval_expr(value))
+        self._env.define(target, self._eval_expr(value))
 
-    def _eval_set(self, name, value):
+    def _eval_set(self, target, value):
-        self._env.assign(name, self._eval_expr(value))
+        match target:
+            case ["index", expr, index]:
+                match [self._eval_expr(expr), self._eval_expr(index)]:
+                    case [["arr", values], int(i)]:
+                        values[i] = self._eval_expr(value)
+                        return
+            case str(name):
+                self._env.assign(target, self._eval_expr(value))
+                return
+        assert False, f"Illegal assignment."
```

多次元配列のときはどうなるのか気になるかもしれませんが、いちばん外側のインデックスだけ特別扱いし、`expr`の中は普通に式として評価すれば大丈夫です。たとえば`a`が`[[5, 6], [7, 8]]`のとき`set a[1][0] = 9`の左辺のASTは`["index", ["index", "a", 1], 0]`となります。まず内側の`["index", "a", 1]`を普通に式として評価すると`a[1]`つまり`[7, 8]`が返されますので、その0番目を更新する、という動きになります。

```
Input source and enter Ctrl+D:
var a = [5, 6, 7]; set a[1] = 8; print a;
Output:
['program', ['var', 'a', ['arr', [5, 6, 7]]], ['set', ['index', 'a', 1], 8], ['print', 'a']]
[5, 8, 7]
Input source and enter Ctrl+D:
var b = [[5, 6], [7, 8]]; set b[1][0] = 9; print b;
Output:
['program', ['var', 'b', ['arr', [['arr', [5, 6]], ['arr', [7, 8]]]]], ['set', ['index', ['index', 'b', 1], 0], 9], ['print', 'b']]
[[5, 6], [9, 8]]
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/arr_assignment/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/arr_reference...arr_assignment)

## 配列用の組み込み関数を実装する

`push()`・`pop()`・`len()`が使えるようにしておきます。選択の理由はなんとなーくそれくらいあるといいかなーというレベル。

```diff py
     def __init__(self):
         self.output = []
         self._env = Environment()
         self._env.define("less", lambda a, b: a < b)
         self._env.define("print_env", self._print_env)
+        self._env.define("push", lambda a, v: a[1].append(v))
+        self._env.define("pop", lambda a: a[1].pop())
+        self._env.define("len", lambda a: len(a[1]))
```

minilangの内部では配列がPythonの配列そのままではなく、`["arr", <配列の実体>]`という形をしているということに気を付けるだけです。

```
Input source and enter Ctrl+D:
var a = [5, 6]; push(a, 7); print a;
print pop(a); print a;
print len(a);
Output:
[...]
[5, 6, 7]
7
[5, 6]
2
```

配列の操作ができるようになったので、アルゴリズムを書ける幅が広がりました。

挿入ソート

```
Input source and enter Ctrl+D:
var a = [3, 7, 6, 4, 9, 2];
var i = 0;
while i < len(a) {
    var v = a[i];
    var j = i;
    while 0 < j {
        if v < a[j - 1] {
            set a[j] = a[j - 1];
        } else {
            break;
        }
        set j = j - 1;
    }
    set a[j] = v;
    set i = i + 1;
}
print a;
Output:
[...]
[2, 3, 4, 6, 7, 9]
```

エラトステネスのふるい

```
Input source and enter Ctrl+D:
var N = 10;
var is_prime = [false, false] + [true] * (N - 2);
var i = 2;
while i * i < N {
    if is_prime[i] {
        var j = i + i;
        while j < N { set is_prime[j] = false; set j = j + i; }
    }
    set i = i + 1;
}
set i = 2;
while i < N {
    if is_prime[i] { print i; }
    set i = i + 1;
}
Output:
[...]
2
3
5
7
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/arr_builtin/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/arr_assignment...arr_builtin)

## for-in文を実装する

C言語風のforを実装するのは気が進まなかったのでPython風のfor-inを実装します。

構文解析は特に注意すべきところはありません。

```diff py
     def _parse_statement(self):
         match self._current_token:
             ...
+            case "for": return self._parse_for()
             ...

+    def _parse_for(self):
+        self._next_token()
+        var = self._parse_primary()
+        assert isinstance(var, str),  f"Expected a name, found `{var}`."
+        self._consume_token("in")
+        array = self._parse_expression()
+        self._check_token("{")
+        body = self._parse_block()
+        return ["for", var, array, body]
```

評価のときは、forで使う変数のためにスコープを作ってやり、配列の要素をひとつひとつ代入しては本体部分を実行してやります。break、continueも使えるようにしておきます。

```diff py
     def _eval_statement(self, statement):
         match statement:
             ...
+            case ["for", var, array, body]: self._eval_for(var, array, body)
             ...
 
+    def _eval_for(self, var, array, body):
+        parent_env = self._env
+        self._env = Environment(parent_env)
+        self._env.define(var, None)
+        for val in self._eval_expr(array)[1]:
+            self._env.assign(var, val)
+            try: self._eval_statement(body)
+            except Continue: continue
+            except Break: break
+        self._env = parent_env
```

動かします。

```
Input source and enter Ctrl+D:
for i in [5, 6, 7] { print i; }
Output:
[...]
5
6
7
Input source and enter Ctrl+D:
for i in [5, 6, 7] { if i = 6 { break; } print i; }
Output:
[...]
5
Input source and enter Ctrl+D:
for i in [5, 6, 7] { if i = 6 { continue; } print i; }
Output:
[...]
5
7
```

`range()`を作ってやれば`for (int i = 0; i < 10; i++) ...`みたいに書く必要はありません。ただしイテレータとかジェネレータではなくてまるごと配列を作ってしまうので効率はよくありません。

```
def range(start, end) {
    var result = []; var i = start;
    while i < end { push(result, i); set i = i + 1; }
    return result;
}

var sum = 0;
for i in range(1, 10) { set sum = sum + i; }
print sum;
```

`map()`・`filter()`・`reduce()`なんかも書きやすく。下の例では配列のうち奇数を取って２倍して合計を計算しています。

```
def map(array, f) {
    var result = [];
    for e in array { push(result, f(e)); }
    return result;
}

def filter(array, f) {
    var result = [];
    for e in array { if f(e) { push(result, e); } }
    return result;
}

def reduce(array, f, init) {
    var result = init;
    for e in array { set result = f(result, e); }
    return result;
}

print
    reduce(
        map(
            filter(
                [5, 2, 3, 9, 6],
                func (e) { return e % 2 = 1; }
            ),
            func (e) { return e * 2; }
        ),
        func (acc, e) { return acc + e; },
        0
    );
```

GitHub: [コード]() [差分]()

## 文字列型を実装する

文字列を導入します。配列も文字列もなにかが並んだものという意味では同じようなものなので同じようにできるところが多い[^due-to]ので一発で導入してしまいます。なお`array`と書いていたところを`seq`やら`index`やらに変えているところがありますがそれだけのところは説明を省略します。

[^due-to]: 直接的にはPythonが配列と文字列を同じように扱ってくれているおかげです。`[]`で何番目かの要素をアクセスできたりfor-inで同じように書けたり。

今回は珍しく字句解析に手が入ります。`"'"`を見つけたら次の`"'"`までを文字列とします。文字列をそのまま返してしまうと識別子と見分けがつかないので配列の時にも使った手で`["str", <文字列>]`という形にして返します。minilangで最初から文字列を扱うことを考えてたら識別子の方を`["id", <名前>]`という形にして文字列はPythonの文字列をそのまま使って表してた、かなあ？

```diff py
     def next_token(self):
         ...
         match self._current_char():
             ...
+            case "'":
+                self._current_position += 1
+                while self._current_char() != "'":
+                    self._current_position += 1
+                self._current_position += 1
+                return ["str", self._source[start + 1:self._current_position - 1]]
             ...
```

構文解析ではたいしたことはしていません。`["str", <文字列>]`というトークンはASTでも`["str", <文字列>]`です。

```diff py
     def _parse_primary(self):
         match self._current_token:
             ...
+            case ["str", s]:
+                self._next_token()
+                return ["str", s]
             ...
```

`["str", <文字列>]`を評価したらただの`<文字列>`になることにします。値になったらもう変数はなくなっているので区別する必要がなくなるため[^hard-decision]。

[^hard-decision]: なにか見落としてないか心配ですが。あと`["str", <文字列>]`の形にしておいた方が配列と同じ形になって統一感を考えるとそっちのほうがいいかなとか、いっそのこと整数型や真偽値型も`["int", 5]`とか`["bool", True]`にしてしまおうかとか。

```diff py
     def _to_print(self, value):
         match value:
             ...
+            case str(s): return s
             ...
 
     def _eval_expr(self, expr):
         match expr:
             ...
+            case ["str", s]: return s
             ...
```

`[]`はほとんど同じように処理できますがパターンだけ追加してやります。

```
-    def _eval_index(self, array, index):
-        match array:
-            case ["arr", value]:
+
+    def _eval_index(self, seq, index):
+        match seq:
+            case ["arr", value] | str(value):
                 return value[index]
             case _:
-                assert False, "Index must be applied to an array."
+                assert False, "Index must be applied to an array or a string."
```

for-inでも同様。

```diff py
-    def _eval_for(self, var, array, body):
+    def _eval_for(self, var, seq, body):
         parent_env = self._env
         self._env = Environment(parent_env)
         self._env.define(var, None)
-        for val in self._eval_expr(array)[1]:
+        match self._eval_expr(seq):
+            case ["arr", values]: seq = values
+            case str(s): seq = s
+        for val in seq:
             self._env.assign(var, val)
             try: self._eval_statement(body)
             except Continue: continue
             except Break: break
         self._env = parent_env
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/str/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/for-in...str)

本日は以上です。
