---
title: "minilangの宿題をやる"
emoji: "🏫"
type: "tech"
topics: ["computerscience", "language", "interpreter", "python", "minilang"]
published: true
---

「[350行くらいのPythonで書くインタプリタ](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines)」の「[拡張のアイデア](https://zenn.dev/kb84tkhr/books/mini-interpreter-in-350-lines/viewer/conclusion#%E6%8B%A1%E5%BC%B5%E3%81%AE%E3%82%A2%E3%82%A4%E3%83%87%E3%82%A2)」にある宿題をやっていきます。順番は適当です。

解説は最小限で行きます。

## コメントを導入する

`!`から行末までをコメントとして扱うようにしました。

```
Input source and enter Ctrl+D:
print 5; ! print 6;
! print 7;
print 8; ! print 9;
Output:
['program', ['print', 5], ['print', 8]]
5
8
```

コード中に`!`を見つけたら改行(`\n`)まで読み飛ばしてから次のトークンを読みに行きます。字句解析で読み飛ばしてしまうので、構文解析以降で手を入れるところはありません。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/comment) [差分](https://github.com/koba925/minilang-book/compare/publish...comment)

## 比較演算子を導入する

minilangの演算子は１文字ということにしているので、`<`と`>`だけ実装します。`<=`と`>=`はありません。そこまでこだわるところでもないんですが。

```
Input source and enter Ctrl+D:
print 5 + 6 < 5 * 6;
Output:
['program', ['print', ['<', ['+', 5, 6], ['*', 5, 6]]]]
true
```

普通に左結合の二項演算子として実装します。優先度は低め。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/comparison) [差分](https://github.com/koba925/minilang-book/compare/comment...comparison)

## 剰余演算子を導入する

`%`で割り算の余りを計算します。比較演算子と剰余のおかげで普通の互除法が書けるようになりました。

```
Input source and enter Ctrl+D:
def gcd(a, b) {
    var tmp = 0;
    while b > 0 {
        set tmp = b; set b = a % b; set a = tmp;
    }
    return a;
}
print gcd(36, 24);
Output:
[...]
12
```

普通に左結合の二項演算子として実装します。優先度はかけ算・割り算と同じにしました。0で割ることはできないので、0割りのチェックを割り算と共有しています。割り算・剰余を引数に入れて渡せるよう、operatorモジュールをimportしています。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/modulus) [差分](https://github.com/koba925/minilang-book/compare/comparison...modulus)

## andとorを導入する

論理積(`and`)、論理和(`or`)を導入します。優先度は最低。

記号にしてもよかったんですが`!`をコメントに使ってしまってたので否定に使ういい感じの記号が残ってなくてPythonに寄せました。

```
Input source and enter Ctrl+D:
print true and false;
Output:
['program', ['print', ['and', True, False]]]
false
```

普通に左結合の二項演算子として実装します。というかできてしまいます。

たとえば`a and b`ではまず`a`を評価し、trueならば`b`を評価する、逆に言うと`a`がfalseなら`b`は見もしない、というショートカットの実装が普通です。`a # 0 and b / a`のように書けば`a`が0でないときだけ`b`を`a`で割るので０割りのエラーにはなりません。関数の評価のように先に左辺も右辺も評価するとこういうチェックができませんし、無駄な計算を行うことになります。

Pythonの`and`は上記のような動作を行いますので、下記の実装では`self._eval_expr(a)`がtrueだったときだけ`self._eval_expr(b)`が行われ、想定通りの動作となります。

```py
            case ["and", a, b]: return self._eval_expr(a) and self._eval_expr(b)
```

明示的に書けばこんな感じの実装になるでしょうか（正確には違います[^and_result]）。

[^and_result]: Pythonの`and`ではTrueやFalseが返るとは限らなくて、`a`が偽なら`a`を、`a`が真ならば`b`を返します。minilangもPythonの`and`をそのまま使っているので同じように動きます。なぜこれでいいのでしょう？

```py
            case ["and", a, b]: return self._and(a, b)
            ...

    def _and(self, a, b):
        if not self._eval_expr(a): return False
        elif self._eval_expr(b): return True
        else: return False 
```

GitHub: [コード](https://github.com/koba925/minilang-book/tree/and_or) [差分](https://github.com/koba925/minilang-book/compare/modulus...and_or)

## notを導入する

否定(`not`)を導入します。Python風にしたので優先度もPythonに寄せて低くしました[^not-priority]。

[^not-priority]: Pythonの`not`の優先度が低いことは今回初めて知りました。JavaScriptなどでは`!`は最高に近い優先度です。たしかに`not`だと低い気がします。なんとなく。

```
Input source and enter Ctrl+D:
print not true;
print not not true;
print not true or true;     
Output:
[...]
false
true
true
```

単項演算子は初出ですが実装は簡単です。右結合なのでべき乗と同じように書けます。再帰しているので`not not true`のように`not`を繰り返して書けます。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/not) [差分](https://github.com/koba925/minilang-book/compare/and_or...not)

## 単項マイナスを導入する

単項演算子を導入したので単項マイナスも導入してしまいましょう。`--a`は`a`を1減らすという意味ではなくて`-(-a)`です。`5--6`も`5-(-6)`になります。

```
Input source and enter Ctrl+D:
print -56;
var a = 5; 
print -a;
print --a;
print 5--6;
print -(a + 3);
Output:
[...]
-56
-5
5
11
-8
```

`not`と単項マイナスは同じ構文ですので`_parse_binop_left`と同様に`_parse_unary`に共通部分をまとめています。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/unary_minus) [差分](https://github.com/koba925/minilang-book/compare/not...unary_minus)

## トップレベルからのreturnをエラーにする

トップレベル（関数の外側）でreturnしたらエラーにします。今まではランタイムエラーでした。

```
Input source and enter Ctrl+D:
return;
Output:
['program', ['return', 0]]
Error: Return at top level.
Input source and enter Ctrl+D:
```

評価器のいちばん外側で`except`してつかまえてあげているだけです。最初からやっておいてもよかったかも。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/return_from_top) [差分](https://github.com/koba925/minilang-book/compare/unary_minus...return_from_top)

## breakとcontinueを導入する

while文の中でbreakとcontinueができるようにします。

```
Input source and enter Ctrl+D:
var n = 5;
while true {
    if n = 8 { break; }
    print n;
    set n = n + 1;
}
print 10;
Output:
[...]
5
6
7
10
Input source and enter Ctrl+D:
set n = 5;
while n < 8 {
    set n = n + 1;
    if n = 7 { continue; }
    print n;
}
print 10;
Output:
[...]
6
8
10
```

whileのなかでやることになっているいろいろをすっ飛ばしてループの外に出たり初めからやり直したりするために、`return`と同様のしくみを使っています。また、スコープを作るときに退避した`_env`を確実に元に戻せるよう、finallyの中に入れました。

GitHub: [コード](https://github.com/koba925/minilang-book/tree/break_continue) [差分](https://github.com/koba925/minilang-book/compare/return_from_top...break_continue)

## nullを導入する

嫌われもののnullですが、optionalとかがあるわけでもないのでやっぱりnullがないとなにか足りない気がします。minilangでは`return;`と返り値を省略したときに使われるくらいです。

```
Input source and enter Ctrl+D:
print null;
print func() { return; }();
Output:
[...]
null
null
```

真偽値を追加したときと同じような作業をします。真偽値はふたつも値があるのにnullはたったひとつなのできっとかんたん！

GitHub: [コード](https://github.com/koba925/minilang-book/tree/null) [差分](https://github.com/koba925/minilang-book/compare/break_continue...null)

## おわりに

今回は以上です。あとは配列あたりをやろうと思ってますがすこし規模が大きくなるので別途。
