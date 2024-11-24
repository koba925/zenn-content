---
title: "minilangに辞書とオブジェクト（のような何か）を追加する"
emoji: "📘"
type: "tech"
topics: ["computerscience", "language", "interpreter", "python", "minilang"]
published: true
---

※「[350行くらいのPythonで作るプログラミング言語実装超入門](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines)」の関連記事です。

minilangで（Python用語でいうところの）辞書を扱えるようにします。それだけなら配列の追加とたいして変わらないのですが、もうちょっといろいろやります。[minilangに配列と文字列を追加する](https://zenn.dev/kb84tkhr/articles/minilang_array_string)の続きです。

やっぱり説明は少なめでいきますので考えたり調べたりしながらどうぞ。特に今回は予備知識がないとわかりづらいかもしれません。質問も歓迎ですのでコメント欄で。
 
## 辞書を実装する

辞書を実装します。やることは配列の実装とたいして変わらないので一気に。

さて、カッコの種類が足りなくなりました。`(`は数式や引数、`{`は複文、`[`は配列に使ってしまっているので。多くの言語で辞書型は`{`で囲まれていますが、そうすると出現個所によっては複文の始まりなのか辞書の始まりなのか判断が付きません。もう少し読んで`:`が出てきたとかそういうところで判断はつけられるのですが構文解析が少々複雑になります。断腸の思いで2文字のトークンにして`$[`と`]`で表すこととします。こちらはたいして複雑になりません。

```diff py
     def next_token(self):
         ...
         match self._current_char():
             ...
+            case "$":
+                self._current_position += 1
+                if self._current_char() == "[":
+                    self._current_position += 1
+                return self._source[start:self._current_position]
             ...
```

構文解析や評価の多くの部分は修正しなくても、またはちょっとした場合分けくらいで動いてくれます。これはPythonが配列も辞書も同じように扱ってくれているおかげ。楽をさせていただいてます。そういったところの説明は省略しますが、気になったらコードを見てなぜ修正なしで配列も辞書も処理できてしまうのか考えてみてください。

以下は辞書リテラルの構文解析です。少しだけ複雑ですが大丈夫でしょう。辞書のキーになる部分はPythonの文字列、つまり変数名が来ることにしています。Pythonではキーに整数も取ることもできて、そのかわり文字列をキーにするときにはクォートで囲む必要があります。このあたりはJavaScriptの方が楽だなと思うところで、識別子が来ることにしています。なので識別子として認められる文字列しかキーにはできません。それ以外は`set foo['$%01'] = 3;`のように`[]`を使って別途追加することになります。

辞書のASTは`["dic", <辞書>]`という形にします。

```diff py
     def _parse_primary(self):
         match self._current_token:
             ...
             case "[": return self._parse_array()
+            case "$[": return self._parse_dic()
             ...
     ...
+    def _parse_dic(self):
+        self._next_token()
+        dic = {}
+        while self._current_token != "]":
+            index = self._current_token
+            assert isinstance(index, str), f"Name expected, found `{index}`."
+            self._next_token()
+            self._consume_token(":")
+            val = self._parse_expression()
+            dic[index] = val
+            if self._current_token != "]":
+                self._consume_token(",")
+        self._consume_token("]")
+        return ["dic", dic]
```

辞書用の組み込み関数`keys()`を追加しておきます。`keys()`があればとりあえずいろいろできそうな気がします。

```diff py
 class Evaluator:
     def __init__(self):
         ...
+        self._env.define("keys", lambda a: ["arr", list(a[1].keys())])
         ...
```

辞書の要素の追加・変更は`set`で行います。`var`は使いません。`foo['bar']`を宣言する、というのは無意味なので。

```diff py
     def _eval_set(self, target, value):
         match target:
             case ["index", expr, index]:
                 match [self._eval_expr(expr), self._eval_expr(index)]:
-                    case [["arr", values], int(i)]:
-                        values[i] = self._eval_expr(value)
+                    case [["arr", values], int(ind)] | [["dic", values], str(ind)]:
+                        values[ind] = self._eval_expr(value)
                         return
             case str(name):
                ...
```

forの評価では辞書の時だけ少し特別扱いして、キーを取り出してからキーで回します。

```diff py
-    def _eval_for(self, var, seq, body):
+    def _eval_for(self, var, col, body):
         parent_env = self._env
         self._env = Environment(parent_env)
         self._env.define(var, None)
-        match self._eval_expr(seq):
-            case ["arr", seq] | str(seq):
-                for val in seq:
-                    self._env.assign(var, val)
-                    try: self._eval_statement(body)
-                    except Continue: continue
-                    except Break: break
+        match self._eval_expr(col):
+            case ["arr", value] | str(value): col = value
+            case ["dic", value]: col = value.keys()
+        for val in col:
+            self._env.assign(var, val)
+            try: self._eval_statement(body)
+            except Continue: continue
+            except Break: break
         self._env = parent_env
```

辞書を文字列化する部分です。配列と同様、内包表記を使って要素をひとつずつ文字列してから`join`でつなぎます。そういえば内包表記の説明はしてなかった気がしますが特殊な使い方でもないので「リスト内包」「辞書内包」あたりで検索していただくということで。キーはやはりクォートしません。

```diff py
     def _to_print(self, value):
         match value:
             ...
+            case ["dic", values]:
+                return "$[" + ", ".join([
+                    self._to_print(key) + ": " + self._to_print(values[key])
+                    for key in values
+                ]) + "]"
             ...
```

辞書の評価では、キーと値それぞれ評価して辞書にします。あと`%`で辞書に要素が含まれているかを判定できるようにしました。`'foo' % $[foo: 1, bar: 2]`のように書いて、`foo`が辞書のキーに含まれているかを判定することができます。Pythonの`in`と同じです。なぜ`%`にしたかというとなんとなくそこにあったからです。優先順位はあんまり適切じゃないかもしれません。すみません。

リスト左端の`>`は「右にインデントしただけだよ」というつもりの記号です。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
+            case ["dic", exprs]:
+                return ["dic", {key: self._eval_expr(exprs[key]) for key in exprs}]
             ...
-            case ["/", a, b]: return self._safe_div_mod(operator.floordiv, self._eval_expr(a), self._eval_expr(b))
-            case ["%", a, b]: return self._safe_div_mod(operator.mod, self._eval_expr(a), self._eval_expr(b))
+            case ["/", a, b]: return self._div_mod_in(operator.floordiv, self._eval_expr(a), self._eval_expr(b))
+            case ["%", a, b]: return self._div_mod_in(operator.mod, self._eval_expr(a), self._eval_expr(b))
             ... 
 
+    def _div_mod_in(self, op, a, b):
+        match b:
+            case ["arr", col] | ["dic", col] | str(col):
+                return a in col
+            case _:
>                assert b != 0, f"Division by zero."
>                return op(a, b)
```

こんな感じになります。

```
Input source and enter Ctrl+D:
print $[a: 1 + 2, b: 3 * 4];
Output:
['program', ['print', ['dic', {'a': ['+', 1, 2], 'b': ['*', 3, 4]}]]]
$[a: 3, b: 12]
Input source and enter Ctrl+D:
print $[a: 5, b: 6]['a'];
Output:
['program', ['print', ['index', ['dic', {'a': 5, 'b': 6}], ['str', 'a']]]]
5
Input source and enter Ctrl+D:
print keys($[a: 5, b: 6]);
Output:
['program', ['print', ['keys', ['dic', {'a': 5, 'b': 6}]]]]
[a, b]
Input source and enter Ctrl+D:
print 'a' % $[a: 5, b: 6];
Output:
['program', ['print', ['%', ['str', 'a'], ['dic', {'a': 5, 'b': 6}]]]]
true
Input source and enter Ctrl+D:
var a = $[a: 5, b: 6]; for k in a { print a[k]; }
Output:
['program', ['var', 'a', ['dic', {'a': 5, 'b': 6}]], ['for', 'k', 'a', ['block', ['print', ['index', 'a', 'k']]]]]
5
6
Input source and enter Ctrl+D:
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/dict/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/str...dict) ※READMEの差分も入ってますがご了承ください

## 辞書の要素をドットで指せるようにする

JavaScriptみたいに、`foo['bar']`を`foo.bar`と書けるようにします。ぐっとシンプルな見た目に。

ASTは同じ形にするので構文解析だけの修正です。ドットの場合は場合分けで。閉じるまで繰り返す、というのがないのでシンプルです。

```diff py
     def _parse_call_index(self):
-        parens = {"(": ")", "[": "]"}
+        parens = {"(": ")", "[": "]", ".": None}
 
         result = self._parse_primary()
         while isinstance((left := self._current_token), str) and left in parens:
             self._next_token()
+            if left == ".":
+                result = ["index", result, ["str", str(self._current_token)]]
+                self._next_token()
+            else:
>                args = []
>                while self._current_token != parens[left]:
>                    args.append(self._parse_expression())
>                    if self._current_token != parens[left]:
>                        self._consume_token(",")
>                if left == "(": result = [result] + args
>                else: result = ["index", result, args[0]]
>                self._consume_token(parens[left])
         return result
```

代入時にもドットで書けるようにします。

```diff py
     def _parse_var_set(self):
         op = self._current_token
         self._next_token()
         target = self._parse_primary()
         assert isinstance(target, str),  f"Expected a name, found `{target}`."
-        while self._current_token == "[":
+        while (left := self._current_token) in ("[", "."):
             self._next_token()
+            if left == "[":
>                index = self._parse_expression()
>                target = ["index", target, index]
>                self._consume_token("]")
+            else:
+                index = self._current_token
+                target = ["index", target, ["str", str(index)]]
+                self._next_token()
         self._consume_token("=")
         value = self._parse_expression()
         self._consume_token(";")
         return [op, target, value]
```

`[]`でも`.`でも同じように書けるようになりました。混じっても大丈夫。

```
Input source and enter Ctrl+D:
var a = $[]; set a.abc = 5; print a; print a.abc; print a['abc'];
Output:
[...]
$[abc: 5]
5
5
Input source and enter Ctrl+D:
var b = $[]; set b.abc = $[]; set b.abc.cde = 5;
print b; print b.abc.cde; print b['abc'].cde; print b.abc['cde'];
Output:
[...]
$[abc: $[cde: 5]]
5
5
5
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/dict_by_dot/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/dict...dict_by_dot)

## 関数をメソッドぽく呼び出す

今でも辞書に関数を覚えさせて呼び出すことはできますが、単に覚えた関数を呼び出すだけです。

```
Input source and enter Ctrl+D:
var a = $[]; set a.abc = func(a) { return 2 * a; }; print a.abc(5);
Output:
[...]
10
```

`set a.double = func(this) { return 2 * this.val; }; print a.double();` とか書けたらちょっとメソッド呼び出しっぽいですね？やってみましょう。

`[]`と`.`で処理が大きく変わるので、`.`のASTを`["dot", ...]`にして別物として扱うことにします。

```diff py
-    def _parse_call_index(self):
+    def _parse_call_index_dot(self):
         ...
         while isinstance((left := self._current_token), str) and left in parens:
             self._next_token()
             if left == ".":
-                result = ["index", result, ["str", str(self._current_token)]]
+                result = ["dot", result, ["str", str(self._current_token)]]
                 self._next_token()
             else:
         ...
```

ほかにも同様な修正箇所がありますが省略。

主役は評価の方です。ドットの右（`index`）が関数だったら、関数の持つ環境にひとつ環境をかぶせてから最初の仮引数（`parameters[0]`）に辞書自身（`seq`）を定義し、最初の仮引数を削ります（`parameters[1:]`）。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
+            case ["dot", expr, index]:
+                return self._eval_dot(self._eval_expr(expr), self._eval_expr(index))
             ...

+    def _eval_dot(self, seq, index):
+        match seq:
+            case ["dic", seq]:
+                match seq[index]:
+                    case ["func", parameters, body, env]:
+                        env = Environment(env)
+                        env.define(parameters[0], ["dic", seq])
+                        return ["func", parameters[1:], body, env]
+                    case _:
+                        return seq[index]
+            case _:
+                assert False, "Dot must be applied to a dic."
```

これで、`foo.bar = func (a, b) {...}`のとき`foo.bar(5)`を評価するとあたかも`bar(foo, 5)`であるかのように評価されるようになります。`a`と書いてあるとそれっぽくないですが`foo.bar = func (this, b) {...}`とか`foo.bar = func (self, b) {...}`と書いて`foo.bar(5)`と呼び出せるわけで、なんかオブジェクトっぽくなってきましたよ！

```
Input source and enter Ctrl+D:
var a = $[val: 5];
set a.double_val = func(this) { return 2 * this.val; };
print a.double_val();
Output:
[...]
10
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/method_like/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/dict_by_dot...method_like)

## プロトタイプチェーン（らしきもの）を実装する

オブジェクトっぽくなってきたとはいえ、まだ単にメソッドぽく呼び出せているだけです。じゃあ`b`でも！と思っても`b`には`double_val`が定義されていないので失敗します。そりゃそうです。

```
Input source and enter Ctrl+D:
var a = $[val: 5];
set a.double_val = func(this) { return 2 * this.val; };
print a.double_val();
var b = $[val: 5];
print b.double_val();
Output:
[...]
Traceback (most recent call last):
  ...
```

そこでまたもやJavaScriptから「プロトタイプチェーン」のアイデアを借りてきます。ざっくり言うとひな形（プロトタイプ）になるオブジェクトを指定しておいて、指定された属性を自分が持っていないときはひな形に同じ属性がないか探しに行く、というものです[^imitation]。これでひな形を継承してるような動作になります。クラスを定義するための構文が不要なので経済的！

[^imitation]: 正直みようみまねレベルなのでちゃんとしたプロトタイプチェーンにはなってないかも・・・チェーンをたどるための最低限のことしかやってないし。逆に、プロトタイプとか考えなくても単にクローンを作るだけでもだいたいやりたいことができるのでは？という気もしています。ただ浅いコピーだと予期しない動きをしそうだし、深いコピーは不経済な気が。教えて有識者！

プロトタイプを辞書の`__proto__`という属性に覚えさせておこうと思うので、字句解析にちょっと手を入れて名前の先頭がアンダースコアでもよいことにします。

```diff py
 class Scanner:
 
     def next_token(self):
         ...
         match self._current_char():
             case "$EOF": return "$EOF"
-            case c if c.isalpha():
+            case c if c.isalpha() or self._current_char() == "_":
                 while self._current_char().isalnum() or self._current_char() == "_":
                     ...
```

その代わり、`__`で始まる属性は`keys()`で列挙しないようにします。

```diff py
 class Evaluator:
     def __init__(self):
         ...
-        self._env.define("keys", lambda a: ["arr", list(a[1].keys())])
+        self._env.define("keys", lambda a: ["arr", [k for k in a[1].keys() if not k.startswith("__")]])
         ...
```

大事なのはドットの評価です。

* 辞書に`index`が含まれていなかったら`__proto__`の指す辞書を探しに行く
* `__proto__`もなければエラー
* 最初に辞書を`this`に覚えておき、`index`の指す値が関数ならひとつめの引数に`this`を割り当てる

ということをやっています。

```diff py
-    def _eval_dot(self, seq, index):
+    def _eval_dot(self, seq, index, this = None):
+        if this is None: this = seq
         match seq:
-            case ["dic", seq]:
-                match seq[index]:
+            case ["dic", dic]:
+                if index not in dic:
+                    assert "__proto__" in dic, f"`{index}` not in the dic."
+                    return self._eval_dot(dic["__proto__"], index, this)
+                match dic[index]:
                     case ["func", parameters, body, env]:
                         env = Environment(env)
-                        env.define(parameters[0], ["dic", seq])
+                        env.define(parameters[0], this)
                         return ["func", parameters[1:], body, env]
                     case _:
-                        return seq[index]
+                        return dic[index]
             case _:
                 assert False, "Dot must be applied to a dic."
```

これだけです。動作確認。

```
Input source and enter Ctrl+D:
var proto = $[val: 5, pr: func(this) { print this.val; }];
var inh1 = $[__proto__: proto]; inh1.pr();
var inh2 = $[__proto__: proto, val: 6]; inh2.pr();
Output:
[...]
5
6
```

ちゃんとプロトタイプを見に行っているようですがこれだけだと継承のイメージがわかないのでもう少し具体的な例を。Personのひな型定義 → Personインスタンスの作成・利用 → Personを継承したChefインスタンスの定義 → Chefインスタンスの作成・利用を行っています。コメントで補足を入れました。

だいぶ長めでプロトタイプになじみがないとわかりづらいかもしれませんが、インタプリタのコードと見比べつつ何をやってるか考えてみてください（説明の放棄）。

```
Input source and enter Ctrl+D:
!
! Personプロトタイプの定義
!

var Person = $[];                    ! プロトタイプの作成

! Personプロトタイプのメソッド
set Person.introduce = func(this) { print 'I am ' + this.name +'.'; };
set Person.move_to = func(this, to) { set this.address = to; };

! Personプロトタイプのコンストラクタ
def new_person(name, address) {
    var this = $[__proto__: Person]; ! ひな形を指定してオブジェクトを作成
    set this.name = name;            ! プロパティの定義
    set this.address = address;      ! プロパティの定義
    return this;
}
Output:
[...]

Input source and enter Ctrl+D:
! Personをひな形にしたインスタンスの作成
var person1 = new_person('Tom', 'Tokyo');

! Personインスタンスの利用
person1.introduce();
print person1.address;
person1.move_to('Osaka');
print person1.address;

Output:
[...]
I am Tom.
Tokyo
Osaka
Input source and enter Ctrl+D:
!
! Chefプロトタイプの定義
!

! プロトタイプの初期化
var Chef = $[__proto__: Person];          ! Personをひな形とする

! メソッド定義
set Chef.cook = func(this) {
    print 'I cooked ' + this.cuisine + ' cuisine.';
};

! コンストラクタ
def new_chef(name, address, cuisine) {
    var this = new_person(name, address); ! Personインスタンスを作成
    set this.__proto__ = Chef;            ! ひな形をChefに変更
    set this.cuisine = cuisine;           ! プロパティの定義
    return this;
}
Output:
[...]

Input source and enter Ctrl+D:
! Chefをひな形にしたインスタンスの作成
var chef1 = new_chef('Jack', 'Paris', 'French');

! Chefインスタンスの利用
chef1.introduce();
print chef1.address;                      ! 継承した属性
chef1.move_to('London');                  ! 継承したメソッド
print chef1.address;                      ! 実行結果の確認
chef1.cook();                             ! Chef独自のメソッド
Output:
[...]
I am Jack.
Paris
London
I cooked French cuisine.
```

GitHub: [コード](https://github.com/koba925/minilang-book/blob/prototype/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/method_like...prototype)

## UFCSを実装する

「プロトタイプ」以上に耳慣れないかもしれませんがUFCS(Uniform Function Call Syntax)と呼ばれる技法があります。`foo(bar, baz)`という関数があったら、`foo(5, 6)`関数の呼び出しの代わりに`5.foo(6)`と書いてもよい、というものです[^dlang-ufcs]。なんかさっきやったのと似てますね？ちょっとしたアイデアに見えますがコードの見た目上大きな効果があります[^pipeline-operator]。やってみましょう。

[^dlang-ufcs]: D言語のドキュメントですが[Uniform Function Call Syntax (UFCS) - Dlang Tour](https://tour.dlang.org/tour/en/gems/uniform-function-call-syntax-ufcs)をご覧ください。

[^pipeline-operator]: 類似でパイプライン演算子というものがありますが、ドットでいいんじゃない？という気がしています（違いがよくわかってない）

ドットの評価方法を変えるだけです。プロトタイプチェーンをたどっても該当する属性が見つからなければ、環境から変数を探し、ユーザ定義関数または組み込み関数であれば最初の引数にドットの左辺の値を割り当てた関数を返してやります。funcの場合はこれまで同様に、builtinの場合は`functools.partial`を使っています。`functools.partial`は、関数の一部の引数に値を割り当てた関数を作るものです。詳しくは[functools.partial — Python 3.11.10 ドキュメント](https://docs.python.org/ja/3.11/library/functools.html#functools.partial)をご覧ください。

また、`_eval_dot()`の中でローカルに`ufcs()`を定義しています。見慣れないかもしれませんが普通に呼び出せますし、`_eval_dot()`内の変数にもアクセスできます。minilangにもあるブロック構造です。変更が多いように見えますが、コードを整理しなおしているだけの部分もあるので落ち着いて読んでください。

```diff py
     def _eval_expr(self, expr):
         match expr:
             ...
-            case ["dot", expr, index]:
-                return self._eval_dot(self._eval_expr(expr), self._eval_expr(index))
+            case ["dot", left, right]:
+                return self._eval_dot(self._eval_expr(left), self._eval_expr(right))
             ...
     
-    def _eval_dot(self, seq, index, this = None):
-        if this is None: this = seq
-        match seq:
+    def _eval_dot(self, left, right, this = None):
+        def ufcs(right):
+            match right:
+                case ["func", parameters, body, env]:
+                    env = Environment(env)
+                    env.define(parameters[0], this)
+                    return ["func", parameters[1:], body, env]
+                case builtin if callable(builtin):
+                    return functools.partial(builtin, left)
+                case _:
+                    return right
+
+        if this is None: this = left
+        match left:
             case ["dic", dic]:
-                if index not in dic:
-                    assert "__proto__" in dic, f"`{index}` not in the dic."
-                    return self._eval_dot(dic["__proto__"], index, this)
-                match dic[index]:
-                    case ["func", parameters, body, env]:
-                        env = Environment(env)
-                        env.define(parameters[0], this)
-                        return ["func", parameters[1:], body, env]
-                    case _:
-                        return dic[index]
+                if right not in dic:
+                    if not "__proto__" in dic:
+                        return ufcs(self._env.get(right))
+                    return self._eval_dot(dic["__proto__"], right, this)
+                return ufcs(dic[right])
             case _:
-                assert False, "Dot must be applied to a dic."
+                return ufcs(self._env.get(right))
```

軽い動作確認から。

```
Input source and enter Ctrl+D:
print keys($[abc: 5, cde: 6]); 
print $[abc: 5, cde: 6].keys();
Output:
[...]
[abc, cde]
[abc, cde]
Input source and enter Ctrl+D:
def f(a) { print a * 2; }
f(5);
5.f();
Output:
[...]
10
10
```

[minilangに配列と文字列を追加する > for-in文を実装する](https://zenn.dev/kb84tkhr/articles/minilang_array_string#for-in%E6%96%87%E3%82%92%E5%AE%9F%E8%A3%85%E3%81%99%E3%82%8B)でやった「配列のうち奇数を取って２倍して合計を計算」の見通しがずいぶんよくなります（printしているところ）。

```
Input source and enter Ctrl+D:
def reduce(array, f, init) {
    var result = init;
    for e in array { set result = result.f(e); }
    return result;
}

def map(array, f) {
    return array.reduce(func(acc, e) {
        acc.push(f(e)); return acc;
    }, []);
}

def filter(array, f) {
    return array.reduce(func(acc, e) {
        if f(e) { acc.push(e); }
        return acc;
    }, []);
}

print [5, 2, 3, 9, 6]
      .filter(func (e) { return e % 2 = 1; })
      .map(func (e) { return e * 2; })
      .reduce(func (acc, e) { return acc + e; }, 0);
Output:
[...]
34
```

Pythonだと`reduce(map(filter()))`的に書いてやらないといけません。UFCSほしい。


## イテレータを書いてみる

同じく[minilangに配列と文字列を追加する > for-in文を実装する](https://zenn.dev/kb84tkhr/articles/minilang_array_string#for-in%E6%96%87%E3%82%92%E5%AE%9F%E8%A3%85%E3%81%99%E3%82%8B)でこんなことを言ってました。

> ただしイテレータとかジェネレータではなくてまるごと配列を作ってしまうので効率はよくありません。

道具立てがそろったのでイテレータを作ってみましょう。`next()`を定義しておけば`foreach()`で各要素について関数を呼ぶことができます。組み込みじゃないのでfor文では使えませんが。Iteratorプロトタイプを作り、それを継承してRangeプロトタイプやListIterプロトタイプを作り、さらにListプロトタイプを作ってListIterを作成できるようにしています。Listプロトタイプのコンストラクタは歴史的な事情で`cons`という名前です。List型でもRange型でもどちらでも同じように`foreach`が呼べています。

あとは読んでみてください！List型の方はちょっと予備知識がないとわかりづらいかもしれませんが。

```
Input source and enter Ctrl+D:
var Iterator = $[];
set Iterator.next = func(this) { return null; };
set Iterator.foreach = func(this, f) {
    var e = null;
    while true {
        set e = this.next();
        if e = null { break; }
        f(e);
    }
};
def iterator() {
    return $[];
}

var Range = $[__proto__: Iterator];
set Range.next = func(this) {
    var result = this._current;
    set this._current = this._current + 1;
    if this._current > this._end { return null; }
    return result;
};
def range(start, end) {
    var this = iterator();
    set this.__proto__ = Range;
    set this._current = start;
    set this._end = end;
    return this;
}

range(3, 6).foreach(func(e) { print e; });

var ListIter = $[__proto__: Iterator];
set ListIter.next = func(this) {
    if this._current = null { return null; }
    var ret = this._current.car;
    set this._current = this._current.cdr;
    return ret;
};
def list_iter(list) {
    var this = iterator();
    set this.__proto__ = ListIter;
    set this._current = list;
    return this;
}

var List = $[];
set List.iter = func(this) { return list_iter(this); };
def cons(car, cdr) {
    var this = $[__proto__: List];
    set this.car = car;
    set this.cdr = cdr;
    return this;
}

var l = cons(3, cons(4, cons(5, null)));
l.iter().foreach(func(e) { print e; });
Output:
[...]
3
4
5
3
4
5
```

Iteratorプロトタイプを作って継承しなくても普通の関数として`foreach()`だけ定義してUFCSとダックタイピングでいけるんじゃ？というツッコミは受け入れます。プロトタイプがちゃんと動いてくれるか心配なので検証を兼ねて使ってみたいのです。

minilangでもずいぶんいろいろ書けるようになりました。ここまでやっても563行。最近の実装はエラー処理が盛大に抜けてますが・・・

GitHub: [コード](https://github.com/koba925/minilang-book/blob/ufcs/minilang.py) [差分](https://github.com/koba925/minilang-book/compare/prototype...ufcs)
