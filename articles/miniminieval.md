---
title: "70行のminilangでつくる評価器超入門（うそ"
emoji: "📘"
type: "tech"
topics: ["computerscience", "language", "interpreter", "minilang"]
published: true
---

※「[350行くらいのPythonで作るプログラミング言語実装超入門](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines)」の関連記事で[minilangに辞書とオブジェクト（のような何か）を追加する](https://zenn.dev/kb84tkhr/articles/minilang_dict_object)の続きです。これまでの記事は[minilangの記事一覧 | Zenn](https://zenn.dev/topics/minilang)をご覧ください。

※※ コードにおかしいところがあったので修正しました。
* 変数名に`var`を使っていてキーワードとまる被りなので`name`に変更しました。動くけど（負け惜しみ
* `eval(expr[2])`の前に`this.`がついてるべきでした。動くけ（略

minilangにもいろいろ機能を追加して、テストは動いてるんだけれども複雑なコードを書いたときにほんとにちゃんと動くのか心配になってきたのでEvaluatorを書いてみました。

数字、真偽値、set、var、if、inc（1を足す）、dec（1を引く）、zero?（0ならtrue）、無名関数、関数呼び出しだけです。サンプルコード。

足し算

```
eval(['var', 'add', ['func', ['a', 'b'],
        ['if', ['zero?', 'b'],
            'a',
            ['add', ['inc', 'a'], ['dec', 'b']]]]]);
```

カウンター

```
eval(['var', 'make_counter', ['func', [],
        [['func', ['count'],
            ['func', [],
                ['set', 'count', ['inc', 'count']],
                'count']],
            0]]]);
```

本題へ。まず、minilangの方でいくつか組み込み関数を増やしました。

* `type` 値の型を返す
* `first` 配列の最初の要素を返す
* `rest` 配列の2番目以降の要素を返す
* `error` プログラムをその場でエラー終了させる

```diff py
 class Evaluator:
     def __init__(self):
         ...
+        self._env.define("type", self._type)
         ...
+        self._env.define("first", lambda a: a[1][0])
+        self._env.define("rest", lambda a: ["arr", a[1][1:]])
         ...
+        self._env.define("error", self._error)
+
+    def _error(self, message): assert False, message
+
+    def _type(self, a):
+        match a:
+            case None: return "null"
+            case bool(_): return "bool"
+            case int(_): return "int"
+            case str(_): return "str"
+            case c if callable(c): return "builtin"
+            case ["func", *_]: return "func"
+            case ["arr", *_]: return "arr"
+            case ["dic", *_]: return "dic"
+            case unexpected: assert False, f"`{unexpected}` unexpected in `_type`."
```

ではminilangで書く部分です。解説はつけていませんがminilangと同じようなことをやってるので雰囲気はわかるのではないかと。実行してみようという奇特な方はREPLにコピペしてください。4つのかたまりに分けてますが一度に入力しても途中でEnterしても問題ありません。

コンストラクタをすこーし楽に書くための関数を作りました。

```
def new(proto, prop) {
    var this = $[__proto__: proto];
    for k in prop { set this[k] = prop[k]; }
    return this;
}
```

環境の操作。だいたい素直に書けてると思います。ここまでついてきていただけた方なら読めるのでは・・・

```
var Environment = $[
    define: func(this, name, val) {
        set this._vals[name] = val;
    },
    assign: func(this, name, val) {
        if name % this._vals { set this._vals[name] = val; }
        elif this._parent # null  { this._parent.assign(name, val); }
        else { error(name + ' not defined.'); }
    },
    get: func(this, name) {
        if name % this._vals { return this._vals[name]; }
        elif this._parent # null { return this._parent.get(name); }
        else { error(name + ' not defined.'); }
    }
];
def environment(parent) {
    return new(Environment, $[_parent: parent, _vals: $[]]);
}
```

評価も同様です。minilangとは違ってこの評価器は値を返します。そのかわりprint文がありません。
分割代入とかパターンマッチとかがあるに越したことはないですがなくてもそこまで書きづらくはありません。

```
var Evaluator = $[
    eval: func(this, expr) {
        if type(expr) = 'int' or type(expr) = 'bool' { return expr; }
        if type(expr) = 'str' { return this._env.get(expr); }

        if expr[0] = 'var' { this._env.define(expr[1], this.eval(expr[2])); return; }
        if expr[0] = 'set' { this._env.assign(expr[1], this.eval(expr[2])); return; }
        if expr[0] = 'if' {
            if this.eval(expr[1]) {
                return this.eval(expr[2]);
            } else {
                return this.eval(expr[3]);
            }
        }
        if expr[0] = 'func' { return [first(rest(expr)), rest(rest(expr)), this._env]; }

        return this._apply(expr);
    },
    _apply: func(this, expr) {
        var op = this.eval(first(expr));
        var args = []; for a in rest(expr) { push(args, this.eval(a)); }
        if type(op) = 'func' { return op(args); }

        var params = op[0]; var body = op[1]; var env = op[2];
        var cur_env = this._env; set this._env = environment(env);
        this._add_args(params, args);
        var ret = null; for expr in body { set ret = this.eval(expr); }
        set this._env = cur_env;
        return ret;
    },
    _add_args: func(this, params, args) {
        if params = [] { return; }
        this._env.define(first(params), first(args));
        this._add_args(rest(params), rest(args));
    }
];
def evaluator() {
    var this = new(Evaluator, $[_env: environment(null)]);
    this._env.define('inc', func(args) { return args[0] + 1; });
    this._env.define('dec', func(args) { return args[0] - 1; });
    this._env.define('zero?', func(args) { return args[0] = 0; });
    return this;
}

var _ev = evaluator();
def eval(expr) { return _ev.eval(expr); }
```

evalが定義できたら動かしてみます。このevalは何もprintせず値を返すだけなので、値をprintしてやる必要があります。

```
Input source and enter Ctrl+D:
print eval(['inc', 5]);
Output:
['program', ['print', ['eval', ['arr', [['str', 'inc'], 5]]]]]
6
```

足し算を定義します。値はないのでprintしてやる必要はありません。ちゃんと定義してくれているはず。

```
Input source and enter Ctrl+D:
eval(['var', 'add', ['func', ['a', 'b'],
        ['if', ['zero?', 'b'],
            'a',
            ['add', ['inc', 'a'], ['dec', 'b']]]]]);
Output:
[...]
```

足し算の実行。こちらは値をprintします。

```
Input source and enter Ctrl+D:
print eval(['add', 5, 6]);
Output:
[...]
11
```

カウンターの定義。

```
Input source and enter Ctrl+D:
eval(['var', 'make_counter', ['func', [],
        [['func', ['count'],
            ['func', [],
                ['set', 'count', ['inc', 'count']],
                'count']],
            0]]]);
Output:
[...]
```

カウンターの作成と実行。

```
Input source and enter Ctrl+D:
eval(['var', 'c1', ['make_counter']]);
eval(['var', 'c2', ['make_counter']]);
Output:
[...]

Input source and enter Ctrl+D:
print eval(['c1']);
print eval(['c1']);
print eval(['c2']);
print eval(['c1']);
print eval(['c2']);
print eval(['c2']);
Output:
[...]
1
2
1
3
2
3
```

どうやらそこそこ複雑なプログラムでもぼろを出さずに動いてくれるようです。ただこれくらいのコードになってくるとデバッグがまあまあ地獄でした。もうちょっとデバッグを支援してくれる機能がほしくなりましたが、エラーの行番号も出さない言語でそこまでやる？というのが躊躇ポイントです。そこら中書き換えることになるので１から書き直したくなったり。

オブジェクトっぽい書き方もテストしておきたかったのでわざわざそういう書き方をしましたが、実は全然必要なくて、たぶんそのために20行くらい増えてます。あと、プロトタイプによる継承はちょっと使いどころが思いつかず。全部Objを継承するようにしたとして、何をやらせたら便利かなあ？

字句解析と構文解析をつけてもうちょっと言語っぽくすることも考えたんですがもう少し文字列の組み込み関数がほしくなるかな？１文字ずつのアクセスはできるからたいていのことは自分で書けそうだけれども？とか考えててまずはEvaluatorを書いてみました（あとデバッグが（以下略

とかいいつつそのうちやるかも。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/miniminieval) [差分](https://github.com/koba925/minilang-book/compare/ufcs...miniminieval)