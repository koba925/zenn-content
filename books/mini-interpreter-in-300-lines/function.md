---
title: "関数"
---

「関数を作る！」って一気に作ってしまうとけっこういっぱい考えないといけないことがあるので
できるだけ細切れにしていきます。

## 組み込み関数と関数呼び出し

`less`という名前で `a < b` なら true、そうでなければfalseを返す組み込み関数を作りましょう。
組み込み関数はpythonのlambdaで表します。
名前が`less`で値が`lambda a, b: a < b`という変数をあらかじめ定義しておきます。

呼び出しは`less(1, 2)`、ASTは`["less", 1, 2]`という形にします。
ASTは丁寧に`["builtin", "less", 1, 2]`などとしてもよいのですがそうしなくてもわかるので。
優先度は高く、primaryのすぐ下になります。
優先度と言ってもちょっとわかりづらいかもしれませんが、
`less(1, 2)`で言うと左辺が`less`、演算子が`(`、右辺が`1,2`、最後のカッコは終わりを示すための単なるしるし、と思えば`+`などと同じように見えてくるでしょう。

ユーザ定義関数型が出てきてからの話になりますが、`some_func_returns_func(2, 3)(4)`のようにも書けますのでそのようにします。
これは左結合です。
まず`some_func_returns_func(2, 3)`を評価して、その結果として返ってきた関数に`4`を引数として与えて呼ぶ、とうことになりますので。
右結合だとするとまず`(2, 3)(4)`を評価して、ということになりますが意味が分かりません。
あとはユーザ定義関数のところで。

`+`などと同様、引数も評価してから渡しますが、関数の実体を取り出すために関数名も評価します。

lessも1やtrueなどと同様、普通に値です。
int、boolに加えて、地味に（？）型がひとつ増えているのです。

print less;

互除法がすんなり書けるようになりました。


## 式文

関数の実装とは直接関係ないのですが、
`print_sum(1, 2);`みたいなのが書けるようにしておきます。
`<式>;`という形です。式文(expression statement)と呼ばれます。

<式>なので`2 + 3;`などとも書けるのですが、2と3を足した後
ただ捨てるだけになってしまい意味がないのでいままで実装していませんでした。

式の先頭にしらないトークンが出てきたらきっと式文に違いないと想定して処理します。

式文を追加しただけだとまだなにもできないので
ちょっとステキな（そこまででもない）組み込み関数を追加しておきます。

読んだ時の環境を表示してくれます。
これから環境の動きがちょっと複雑になってわかりにくくなってきます。
なんか動きがおかしいなと思ったら読んでみましょう。

## func型の追加

組み込み関数ができましたので、次はいよいよユーザ定義関数を書けるようにします。
defなしreturnなし

defなしでどうするのかというと、無名関数（Pythonで言うところの`lambda`）だけ実装します。
`lambda a: print(a)`は`func(a) { print a; }`と書きます。
`def pr(a): print(a)`に当たる書き方は`var pr = func(a) { print a; }`です。
呼ぶときは普通に`pr(3);`などと呼べます。

JavaScriptやっているひとにはおなじみかもしれませんが、名前は必須ではなく、名前を付けずに呼ぶこともできます。
`func(a) { print a; }(3)`とすると、`{func(a) { print a; }}`という関数に`3`という引数を与えて実行します。
`func(){}`のほうが`(3)`よりも優先順位が高いということでもあります。
`func`は最後の`}`まで読んで初めて値になるので、`primary`の仲間になります。

returnはあとから実装します。
returnが書いてなければ0を返すことにします。

`def _apply(self, [parameters, body, env], args)`って書きたい！

組み込み関数と同じように、ひとつ型を追加する感じになります。
まずは「ユーザ定義関数型」を書けるようにしましょう（実行するのは後、ということ）。

var sum = func(a, b) { var c = 4; print a + b; print_env(); }; sum(2, 3);

## func型の呼び出し・実行

呼び出しは組み込み関数と同じ

使い捨ての環境を作って、与えられた引数を普通の変数と同様にして定義してやります。
その環境の中で関数本体を実行すれば、引数が与えられた状態で実行されるというわけです。
終わったら環境を捨てます。

すでにいくらかインタプリタを勉強されてる方は「アレ？」と思っているかもしれませんがしばしお待ちを。

## return

環境捨てるところすっとばして抜けちゃっていいの？と思われるかもしれませんが、見てる人がいなくなるのでいずれにしろガベージコレクションされるはずです。たぶん。


関数を返す関数も作れます。
組み込みでもユーザ定義でも。

## 動的スコープと静的スコープ（またはレキシカルスコープ）のお話



`func(a) { return a + b; };`

Input source and enter Ctrl+D:
print func(a) { return a + b; } (5);
Output:
Error: `b` not defined.

まあそりゃそうですよね

では、こういうのだったら？

var b = 6;
print func(a) { return a + b; } (5);

成功します

Input source and enter Ctrl+D:
var b = 6;
print func(a) { return a + b; } (5);
Output:
11

これは、今のminilangは今見てる環境で見つからなかったら呼び出し元の環境を見に行くためです。
最上位の環境にあるbが見えています。

Input source and enter Ctrl+D:
var b = 6;
print func(a) { print_env(); return a + b; } (5);
Output:
{'less': '<builtin>', 'print_env': '<builtin>', 'b': 6}
{'a': 5}
{}
11

これは？

var fa = func(a) { print_env(); return a + b; };
var b = 6;
print fa(5);

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



## def

* 関数を定義する文を作る
  `let add = func(a, b) { ... }`などと書く代わりに`def add(a, b) { ... }`と書けるようにします。文の先頭で`func`を見つけたら`var`の形に直してASTを作ればOKです。Evaluatorは変更不要です。
  シンタックスシュガー

elifでも似たようなことをしましたね

def fib(n) {
    if n = 1 { return 1; }
    if n = 2 { return 1; }
    return fib(n - 1) + fib(n - 2);
}
print fib(6);

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

おお、なんか普通のプログラミング言語っぽいです（そういう題材を選んだだけとも言う

343行！