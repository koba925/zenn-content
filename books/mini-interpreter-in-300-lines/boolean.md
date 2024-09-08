---
title: "真偽値を実装する"
---

真偽値（Boolean、bool）を作りましょう。
ミニ言語的には0を偽、それ以外を真とみなすとかして省略するっていうのもありですが、if文やwhile文を作るときにストレートに書けるのと、型を追加する作業っていうのも入ってた方がいいかなということで普通に真偽値を追加することにしました。

　`"123"`という文字列を`123`という数に変換していたように、`"true"`という文字列をPythonの`True`に変換してやり、あとは　今まで数が入っている前提で処理していたところで、型を見て処理するようにします。型が合わないところではエラーにすることも。
Pythonのboolを使います。

今回は`Scanner`にも手が入ります。
名前を切り出したらそのまま返さず、`"true"`という文字列だったら`True`を、`"false"`だったら`False`を返すようにします。

```diff py
 class Scanner:
     ...

     def next_token(self):
         while self._current_char().isspace(): self._current_position += 1
 
         start = self._current_position
         match self._current_char():
             case "$EOF": return "$EOF"
             case c if c.isalpha():
                 while self._current_char().isalnum() or self._current_char() == "_":
                     self._current_position += 1
-                return self._source[start:self._current_position]
+                token = self._source[start:self._current_position]
+                match token:
+                    case "true": return True
+                    case "false": return False
+                    case _: return token
             case c if c.isnumeric():
                 ...
```

`Parser`では`_parse_primary()`で`int`同様に`bool`を受け付けます。
`True`や`False`をそのまま返していますので、ASTでも`True`や`False`が使われます。
新たな構文を導入するわけではないので、修正はここだけです。

```diff py
 class Parser:
     ...

     def _parse_primary(self):
         match self._current_token:
             ...
-            case int(value):
+            case int(value) | bool(value):
                 self._next_token()
                 return value
             ...
```

`Evaluator`では真偽値の値を出力できるように、値の型を見て文字列を返します。

```diff py
 class Evaluator:
     ...
     def _eval_print(self, expr):
-        self.output.append(self._eval_expr(expr))
+        self.output.append(self._to_print(self._eval_expr(expr)))
+
+    def _to_print(self, value):
+        match value:
+            case bool(b): return "true" if b else "false"
+            case _: return value

```

型そのものの追加はこれで終わりです。
いったんここで区切りをつけて実行してみましょう。

```
Input source and enter Ctrl+D:
print true;
Output:
['program', ['print', True]]
true
Input source and enter Ctrl+D:
print false;
Output:
['program', ['print', False]]
false
```

想定通りです。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/5010_boolean) [差分](https://github.com/koba925/minilang-book/compare/4050_power...5010_boolean)

## 等号

真偽値が書けるようになっただけでは何もできませんので真偽値の演算を導入します。
minilangの演算子は１文字ですので、`=`と`#`[^not-equal]で値が等しいか（等しくないか）を調べることにします。

[^not-equal]: ≠に見えますね？

あまり使い勝手はよくありませんが普通に左結合で実装します[^common]。
ということは、`_parse_binop_left()`がそのまま使えるということです。

[^common]: Pythonのように`1 == 1 == 1`が`true`になったりする便利仕様ではないですよ、ということです。
`1 = 1 = 1`はまず左側の1と1を比較して異なるので`true`、次にその`true`といちばん右の1を比較して、全体として`false`となります。
`1 = 1 = true`なら`true`になります。`(1 = 1) = true`と思えば自然です。

```diff py
     def _parse_expression(self):
-        return self._parse_add_sub()
+        return self._parse_equality()
 
+    def _parse_equality(self): return self._parse_binop_left(("=", "#"), self._parse_add_sub)
     def _parse_add_sub(self): return self._parse_binop_left(("+", "-"), self._parse_mult_div)
     def _parse_mult_div(self): return self._parse_binop_left(("*", "/"), self._parse_power)
 
     def _parse_binop_left(self, ops, sub_element):
         result = sub_element()
```

共通部分を抜き出してあったおかげで少しの修正で済みました。

`Evaluator`の修正はシンプルです。

```diff py
     def _eval_expr(self, expr):
         match expr:
             case int(value) | bool(value): return value
             case ["^", a, b]: return self._eval_expr(a) ** self._eval_expr(b)
             case ["*", a, b]: return self._eval_expr(a) * self._eval_expr(b)
             case ["/", a, b]: return self._div(self._eval_expr(a), self._eval_expr(b))
             case ["+", a, b]: return self._eval_expr(a) + self._eval_expr(b)
             case ["-", a, b]: return self._eval_expr(a) - self._eval_expr(b)
+            case ["=", a, b]: return self._eval_expr(a) == self._eval_expr(b)
+            case ["#", a, b]: return self._eval_expr(a) != self._eval_expr(b)
             case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

型チェックをするならこのあたりで手を入れるところですが、minilangはそのあたりはおおらかな態度でいきます。
自前の型チェックは実装せず、Pythonがどうするかに任せています。
というわけで真偽値同士の`+`とか数値と真偽値の`-`とかがどうなるかはPythonの`+`や`-`がどうなるかにかかっています[^strange-answer]。
独自の仕様にしたい場合は`type(a)`と`type(b)`を見てエラーにしたり異なる型の値に変換したりすることになります。

[^strange-answer]: だから、`print true + true;`とかしてびっくりしても私のせいではないですよ！章末の余談参照。

では。

```
Input source and enter Ctrl+D:
print 3 * 4 = 5 + 6;
Output:
['program', ['print', ['=', ['*', 3, 4], ['+', 5, 6]]]]
false
Input source and enter Ctrl+D:
print 3 * 4 # 5 + 6;
Output:
['program', ['print', ['#', ['*', 3, 4], ['+', 5, 6]]]]
true
Input source and enter Ctrl+D:
print true = false;
Output:
['program', ['print', ['=', True, False]]]
false
Input source and enter Ctrl+D:
print 1 = 1 = true;
Output:
['program', ['print', ['=', ['=', 1, 1], True]]]
true
```

優先順位、結果とも想定通りです。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/5020_equality) [差分](https://github.com/koba925/minilang-book/compare/5010_boolean...5020_equality)

## 余談：

> `print true + true;`とかしてびっくりしても私のせいではないですよ！

型チェックしてませんし型が合わなくてランタイムエラーですかね。
ふつうそう思いますよね。

```
Input source and enter Ctrl+D:
print true + true;
Output:
['program', ['print', ['+', True, True]]]
2
```

えっ？
でも私のせいではありません。

```
>>> True + True
2
```

知ってる人は知ってると思いますがPythonのTrueはintなんです。

``` py
>>> isinstance(True, int)
True
```

ええええ？
Pythonではboolがintのサブクラスだからなんですが。

```py
>>> issubclass(bool, int)
True
```

JavaScriptでもそうなります。

```js
> true + true
2
```

ちなみにFalseは0です。

```py
>>> 1 * False
0
```

Trueが-1になる処理系もあります。
どういう仕様にするかは言語設計者であるあなた次第です[^pep285]。

[^pep285]: Pythonにboolが導入されたときの考え方は https://peps.python.org/pep-0285/ で読めます。
minilangも、はじめのうちは最初は真偽値を実装しないで0をfalse、他はtrueという扱いにしてたんですが、trueを返したいっていうときは1を返すようにしていました。

真偽値どうしの`+`を`or`、真偽値どうしの`*`を`and`と考えたとき、Falseが0のほうがなじみがいいのです。
さらにさかのぼるとブール代数という数学の世界に入ります。

あまりそういう言語は見かけませんが、minilangでは演算子を1文字としているので、`+`を`or`、`*`を`and`として実装するのもよさそうです[^shortcut]。
では真偽値と数値の`+`はどうするか？これはエラー？

[^shortcut]: `or`と`and`は一見ただの二項演算子ですが、実は制御構造っぽいところもあるので注意が必要です。
あまりそういう言語は見かけないというのも、そのあたりの違いを意識させるためかもしれません。
といっても`or`と`and`を使えば特に意識してなくてもそうなりますが。

反対に、if文やwhile文などで真偽値が期待されているところに他の型の値が来たらどうするかっていうのも言語処理系の考え方が出るところです。
真偽値しか認めず他の型はすべてエラーにする言語と、なにかしらのルールで真偽値に変換する言語があり、真偽値に変換する場合も何を真・何を偽とするかで違いが出てきます。

Pythonは、`0`や`""`、`[]`、`{}`など「空っぽ」とみなせるものは偽、他の値は真というルールです。
これはブール代数との整合性もありますが、（空っぽになるまで）ループするときに便利というのもあると思います。
それなりにわかりやすい考え方だと思います。

私自身はそういうルールを思い出してどっちだったっけ？と考えるのが好きではないので明示的に書きたがる方で、「aが0かどうか」を判定するのに`if a:`や`if not a:`と書くのは好きではないのですが、「aが空になるまで」という意味合いの時には`while a:`と書くことも多いです。


