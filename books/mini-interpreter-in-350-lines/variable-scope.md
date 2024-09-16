---
title: "変数"
---

minilangに変数を導入します。
変数の名前のことをこれからは「識別子」(Identifier)と呼びます。

変数の宣言は`var <識別子> = <式>;`、変数の代入は`set <識別子> = <式>;`という文で行うことにします。
変数の代入は<式>の中でできる言語が多いですが、minilangでは処理系のシンプルさを優先してset文を導入しました。
<式>の中で変数を参照するときはもちろん<識別子>を書くだけです。

<識別子>のために新たな形のトークンやASTは導入せず、`print`と同様に単なる文字列で表します。
式のAST内で裸の文字列があったら変数（の名前）とみなす、ということです。
`3 + a`なら`["+", 3, "a"]`となります。
ただし、変数の型だけでは`print`のようなキーワードと見分けがつかないので、文字列自体を見て区別をつける必要があります。
変数の宣言は`["var", <識別子>, <式>]`、変数の代入は`["set", <識別子>, <式>]`の形とします。

## 環境

変数を導入するには、どの変数がどんな値かを覚えておく場所が必要になります。
この場所を「環境」（Environment）と呼びます。

環境のために`Environment`クラスを作ります。
変数は辞書に覚えることにします。
まだスコープを実装してないので、グローバル変数しかありません。

```diff py
 class Parser:
     ...

+class Environment:
+    def __init__(self):
+        self._values = {}
```

次は変数の宣言（定義）に当たる処理を作ります。
すでに定義済みの識別子だったらエラーにして、そうでなければ辞書に名前と値を記録します。

```diff py
+    def define(self, name, value):
+        assert name not in self._values, f"`{name}` already defined."
+        self._values[name] = value
```

つぎは代入です。
宣言とは逆に、定義済みの識別子であれば値を更新して、そうでなければエラーにします。

```diff py
+    def assign(self, name, value):
+        if name in self._values: self._values[name] = value
+        else: assert False, f"`{name}` not defined."
```

最後は値の取得です。
定義済みの識別子であれば値を返し、そうでなければエラーにします。

```diff py
+    def get(self, name):
+        if name in self._values: return self._values[name]
+        assert False, f"`{name}` not defined."

 class Evaluator:
     ...
```

あとでもう少し複雑になりますが、今はこれで。

GitHub: ふるまいはなにも変わっていないのでまだcommitはありません。次の節で。

## 定義・代入・参照

定義・代入・参照をいっぺんに導入します。
どれかひとつだけ導入してもあまり面白くないので。

`Parser`クラスを修正します。
定義と代入は`var`と`set`が違うだけで同じ構文なので、まとめて構文解析します。

```diff py
     def _parse_statement(self):
         match self._current_token:
+            case "var" | "set": return self._parse_var_set()
             case "print": return self._parse_print()
             case unexpected: assert False, f"Unexpected token `{unexpected}`."
 
+    def _parse_var_set(self):
+        op = self._current_token
+        self._next_token()
+        name = self._parse_primary()
+        assert isinstance(name, str),  f"Expected a name, found `{name}`."
+        self._consume_token("=")
+        value = self._parse_expression()
+        self._consume_token(";")
+        return [op, name, value]
```

`op`に`var`か`set`かを覚えておいて後で使います。
minilangでは`var`や`set`の左辺（代入される側）には<識別子>、つまりただの文字列しか来ないので実は`_parse_primary()`を呼ばなくても読んだトークンが文字列かどうか判定するだけでいいのですが、<識別子>は`_parse_primary()`で読むことにしているので`_parse_primary()`を読んで処理しています。
配列や辞書、オブジェクトなどのある言語は左辺がもっと複雑になるので、ただトークンを読めばいいということはなくなります。

その`_parse_primary()`です。

```diff py
     def _parse_primary(self):
         match self._current_token:
             ...
-            case int(value) | bool(value):
+            case int(value) | bool(value) | str(value):
                 self._next_token()
                 return value
             case unexpected: assert False, f"Unexpected token `{unexpected}`."
```

というわけでトークンが文字列だったらそのまま返しているだけです。

次は`Evaluator`の修正です。

```diff py
 class Evaluator:
     def __init__(self):
         self.output = []
+        self._env = Environment()
```

`Environment`オブジェクトを作って覚えておきます。
REPLで`Parser`は毎回つくるのに`Evaluator`は最初に１回作るだけにしていたのはここに意味があります。
`Evaluator`を残しておけば`Environment`も使いまわしになるので、前回作った変数が消えることなく残っているというわけです。

次はvar文、set文の評価です。
文の種類が増えたので`_eval_statement()`に手が入ります。

```diff py 
     def _eval_statement(self, statement):
         match statement:
+            case ["var", name, value]: self._eval_var(name, value)
+            case ["set", name, value]: self._eval_set(name, value)
             case ["print", expr]: self._eval_print(expr)
             case unexpected: assert False, f"Internal Error at `{unexpected}`."
 
+    def _eval_var(self, name, value):
+        self._env.define(name, self._eval_expr(value))
+
+    def _eval_set(self, name, value):
+        self._env.assign(name, self._eval_expr(value))
```

`"var"`だったり`"set"`だったりしたらそれに応じて`Environment.define()`や`Environment.assign()`を読んで環境に覚えておくだけです。

パターンマッチのおかげですっきり書けています。
メソッドに分割する必要もないくらいですが、文法にしたがって機械的に書いてる雰囲気を醸し出そうという気持ちで分割しました。

当然<式>の処理にもにも手が入ります。
<式>の中に文字列が現れたら<識別子>なので変数として扱います。
やることは単純です。

```diff py 
     def _eval_expr(self, expr):
         match expr:
             case int(value) | bool(value): return value
+            case str(name): return self._eval_variable(name)
             case ["^", a, b]: return self._eval_expr(a) ** self._eval_expr(b)
             ...
 
     def _div(self, a, b):
         ...
 
+    def _eval_variable(self, name):
+        return self._env.get(name)

 if __name__ == "__main__":
```

ではやってみましょう。
文が増えるのは初なので少し新鮮な気持ちです。

```
Input source and enter Ctrl+D:
var a = 5 + 6; var b = 7 * 8; print a + b;
var c = 5; print c; set c = c + 6; print c;
var d = true; print d; set d = false; print d;
Output:
['program', ['var', 'a', ['+', 5, 6]], ['var', 'b', ['*', 7, 8]], ['print', ['+', 'a', 'b']], ['var', 'c', 5], ['print', 'c'], ['set', 'c', ['+', 'c', 6]], ['print', 'c'], ['var', 'd', True], ['print', 'd'], ['set', 'd', False], ['print', 'd']]
67
5
11
true
false
Input source and enter Ctrl+D:
var e = 1; var e = 1;
Output:
['program', ['var', 'e', 1], ['var', 'e', 1]]
Error: `e` already defined.
Input source and enter Ctrl+D:
set f = 1;
Output:
['program', ['set', 'f', 1]]
Error: `f` not defined.
Input source and enter Ctrl+D:
print g;
Output:
['program', ['print', 'g']]
Error: `g` not defined.
```

同じ変数を２回定義することができなかったり、存在しない変数に代入しようとするとエラーになったりすることも確認できました。
REPLで使うときは同じ変数を繰り返し定義できたほうが使いやすいのでそうしている処理系もありますがminilangではしていません。
REPL的にはだいぶ使い勝手悪いのですがシンプルさ優先、実用性は二の次で！

GitHub: [コード](https://github.com/koba925/minilang-book/tree/6010_variable) [差分](https://github.com/koba925/minilang-book/compare/5020_equality...6010_variable)

## 複文とスコープ

複文というのは複数の文をまとめてひとつの文として扱えるようにしたものです。
`{ var a = 1; print a; }`といった書き方ができるようになります。
ブロックとも呼ばれます。

複文の中は新たなスコープになるのが一般的です。
複文の中だけで有効なローカル変数が書けるようになります。
スコープには以下のような性質があります。

* スコープ内で定義された変数はスコープを抜けると変数が参照できなくなる。`{ var a = 1; } print a;`はエラー。
* スコープの外側で定義された変数にはアクセスできる。`var a = 1; { print a; }`は`1`を出力する。
* 外側で定義された変数と同じ名前で変数を作ると、内側で作った方にアクセスする。`var a = 1; { var a = 2; print a; } print a;`は`2`・`1`と出力する。

ブロック構造とも呼ばれます。

今の時点で複文・スコープだけあってもそんなにうれしくはないんですが、制御構造や関数の実装に使いたいので変数を導入した今がタイミングかと。

`Parser`に複文を導入します。

```diff py
     def _parse_statement(self):
         match self._current_token:
+            case "{": return self._parse_block()
             case "var" | "set": return self._parse_var_set()
             ...
 
+    def _parse_block(self):
+        block: list = ["block"]
+        self._next_token()
+        while self._current_token != "}":
+            block.append(self._parse_statement())
+        self._next_token()
+        return block
```

`}`が出てくるまで<文>を読みこんで返します。
`parse_program()`と同じようなことをしています。
そろそろパターンが見えてきたでしょうか？

スコープを導入するには`Environment`がキモになります。
さきほどはひとつの辞書に変数を覚えさせていましたが、スコープを実現するには複数の辞書を親子関係でつなぎ、階層構造を持たせます。
新しくスコープを作るときには新しい辞書を作って、元のスコープの子として連結します。
変数の代入や参照の際には階層の最下層から変数を探し始め、その階層に目当ての変数がなければ親の階層で探す、それでもなければさらにその親、と階層を登りつつ探していきます。

ここと、続く`Evaluator`の実装はわかりにくいところのひとつかと思います。
スコープの話は関数の実装でもう一度出てきますのでゆっくり消化しておきましょう。

```diff py
 class Environment:
-    def __init__(self):
+    def __init__(self, parent:"Environment | None"=None):
         self._values = {}
+        self._parent = parent
```

オブジェクトに親の環境を覚えておけるようにします。
`Environment()`と親を指定せずに環境を作ると、`_parent`には`None`が入り、最上位の環境であることを示します。

親環境を指定しながら`Environment`を作っていくと、以下のようにつながった階層状の環境ができていきます。
右の方が上位で、変数を探すときには左の方から探します。

```
│Environment _parent─┼─>│Environment _parent─┼─>│Environment _parent─┼─>None
```

なお`:"Environment | None"`も警告除けの型ヒントです。

```diff py 
     def define(self, name, value):
         assert name not in self._values, f"`{name}` already defined."
         self._values[name] = value
```

変数の定義は常に今いるスコープ、つまり最下層のスコープで行いますのでなにも変更はありません。

```diff py 
     def assign(self, name, value):
         if name in self._values: self._values[name] = value
+        elif self._parent is not None: self._parent.assign(name, value)
         else: assert False, f"`{name}` not defined."
 
     def get(self, name):
         if name in self._values: return self._values[name]
+        if self._parent is not None: return self._parent.get(name)
         assert False, f"`{name}` not defined."
```

代入と参照には「その階層に目当ての変数がなければ親の階層で探す」コードを追加しました。
`_parent`が`None`のときは最上位の階層にいるということですので、そこでも見つからなければエラーで終了です。
ここでも再帰が出てきましたが、「その階層に目当ての変数がなければ親の階層で探す」をそのまま書いているだけですので難しく考える必要はありません。

```diff py
     def _eval_statement(self, statement):
         match statement:
+            case ["block", *statements]: self._eval_block(statements)
             case ["var", name, value]: self._eval_var(name, value)
             ...

+    def _eval_block(self, statements):
+        parent_env = self._env
+        self._env = Environment(parent_env)
+        for statement in statements:
+            self._eval_statement(statement)
+        self._env = parent_env
```

`parent_env`に現在の環境をいったん覚えておいてから、新しく環境を作ります。
このとき、`parent_env`を親環境として指定していますので、親子の階層ができます。
新しく作った環境の中で文を評価していきます。
すべての文を評価し終わったら、環境を元に戻します。
さきほど作成した環境は単に捨てられます。

これで複文とスコープが実装できました。
実行してみます。

```
Input source and enter Ctrl+D:
{ var a = 1; } print a;
Output:
['program', ['block', ['var', 'a', 1]], ['print', 'a']]
Error: `a` not defined.
Input source and enter Ctrl+D:
var b = 5; { print b; }
Output:
['program', ['var', 'b', 5], ['block', ['print', 'b']]]
5
Input source and enter Ctrl+D:
var c = 5; { set c = 6; print c; } print c;
Output:
['program', ['var', 'c', 5], ['block', ['set', 'c', 6], ['print', 'c']], ['print', 'c']]
6
6
Input source and enter Ctrl+D:
var d = 5; { var d = 6; print d; } print d;
Output:
['program', ['var', 'd', 5], ['block', ['var', 'd', 6], ['print', 'd']], ['print', 'd']]
6
5
```

どうでしょう？
環境の階層が積み重なる感じをイメージしつつ実行したりコード読み返したりしてみてください。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/6020_scope) [差分](https://github.com/koba925/minilang-book/compare/6010_variable...6020_scope)