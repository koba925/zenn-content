---
title: "print文だけの言語を作る"
---

最初はprint文と数だけの言語を作ります。
`print 123;`を実行すれば`123`と表示される、それだけ、という言語です。

```
print 1;
print 12;
print 123;
```

というソースを実行すると

```
1
12
123
```

と表示される、ということです。
文章にして書くとこういう言語です。

* `<プログラム>`は`<文>`が続いたもの
* `<文>`は`print <式>;` という形
* （今のところ）`<式>`は`<数>`という形（だけ）
* `print`、`<数>`、`;`の前後は任意の空白を入れてよい
* `<数>`は数字が続いたもの
* print文は`<数>`を表示する

これだって言語。

名前がないと説明しづらいのでこの本で作っていく言語を「minilang」と呼ぶことにします。

## 最初の実装

とにかく書いてみましょう。
上に書いた「こういう言語です」を


```py
import sys, re

def interpret(source):
    statement_pattern = re.compile(r"\s*print\s*(\d+)\s*;\s*")

    current_position = 0
    while current_position < len(source):
        statement = statement_pattern.match(source, current_position)
        assert statement is not None, "Does not match."
        print(int(statement.group(1)))
        current_position = statement.end()

while True:
    print("Input source and enter Ctrl+D:")
    if (source := sys.stdin.read()) == "": break

    print("Output:")
    try: interpret(source)
    except AssertionError as e: print("Error: ", e)
```

これくらいのコードで書けますよ、というだけの例で、ここから先の話にはあまりつながってこないので


tokenって何ですか？と思われた方もいるかもしれませんが後で説明しますので今はあまりお気になさらず。


動かしてみます。

```
$ python minilang.py <Enter>
: print 1; <Enter>
1
```

ちゃんと実行されました。
空白を入れても大丈夫です。

```
$ python minilang.py <Enter>
:    print   123   ;    <Enter>
123
```

入力ソースコード[^input]に誤りがあると実行しません。

[^input]: 本書では、minilangで書かれたプログラムのソースコードと、minilangを実行するプログラムのソースコードを区別するため、前者を「入力ソースコード」と呼ぶことにします。

```
$ python minilang.py <Enter>
: prin 1; <Enter>
Traceback (most recent call last):
  ...
AssertionError: Error occurred.
```

正規表現[^regular-expression]って便利ですね！

[^regular-expression]: `r"^\s*print\s*(\d+)\s*;\s*$"`といった正規表現はひとつの言語とみなすことができます。PythonやJavascriptなどのようにいろいろなことができる言語ではなく、文字列がある形になっているかを判定するという目的にのみ使われる言語なのでDSL (Domain Specific Language) と呼ばれます。目的を絞っているのでその目的に対してはうまく書けるようになっています。正規表現っていう言葉、プログラミングでよく出てくるからと何げなく使っている人も多いと思いますが、計算機科学の世界ではよく研究された分野です。「正規表現」「オートマトン」「形式言語」あたりのことばで検索してみるといろいろ出てきます。

## 自前で

与えられたソースコードの文字列を１文字ずつ見ながらひとかたまりずつ処理していきます。
少々（かなり）冗長ですが全ソース。

https://github.com/koba925/minilang/blob/print/minilang.py

```py
class Interpreter:
    def __init__(self, source):
        self._source = source
        self._current_position = 0

    def interpret(self):
        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        while  self._current_char().isalpha(): self._current_position += 1
        command = self._source[start:self._current_position]
        assert command == "print", f"`print` expected."

        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        while  self._current_char().isnumeric():
            self._current_position += 1
        assert self._source[start:self._current_position].isnumeric(), f"Number expected."
        number = int(self._source[start:self._current_position])

        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        self._current_position += 1
        assert self._source[start:self._current_position] == ";", f"Semicolon expected."

        while self._current_char().isspace(): self._current_position += 1
        assert self._current_char() == "$EOF", f"EOF expected."

        print(number)

    def _current_char(self):
        if self._current_position < len(self._source):
            return self._source[self._current_position]
        else:
            return "$EOF"

if __name__ == "__main__":
    while source := "\n".join(iter(lambda: input(": "), "")):
        try: Interpreter(source).interpret()
        except AssertionError as e: print(e)
```

さっそく動かしてみます。
": "がプロンプトなので、上記文法にしたがって、あるいは従わずに文を入力します。
複数行入力できるようにしました。
空行を受け取ると実行します（Enterを2回押す）。

ちょうどいいところまで入力したら自動的に実行してくれたりしたらカッコいいんですけど、そこまでしてません。
最初から何も入力せずにEnterを2回押すか、エラーが出ると終了します。

```txt
$ python minilang.py 
: print 1; <Enter>
: <Enter>
1
:    print   123   ;    <Enter>
: <Enter>
123
: print <Enter>
: 12345 <Enter>
: ; <Enter>
: <Enter>
12345
: <Enter>
vscode@d88b5abb828d:/workspaces/minilang$ python minilang.py 
: trinp 1; <Enter>
: <Enter>
Traceback (most recent call last):
  File "/workspaces/minilang/minilang.py", line 38, in <module>
    Interpreter(src).interpret()
  File "/workspaces/minilang/minilang.py", line 11, in interpret
    assert command == "print"
           ^^^^^^^^^^^^^^^^^^
vscode@d88b5abb828d:/workspaces/minilang$ 
```

ではソースコードをパーツごとに見ていきます。

ソースコードとソースコード上の現在位置を覚えておきます。
`_source`や`_current_position`の前についているアンダースコアは、
この属性はクラスの中だけで使いますよ、他からは使わないでね、というしるしです[^underscore]。

[^underscore]:ただのしるしなので、使えば使えてしまいます。
アンダースコアをふたつ並べれば（簡単には）使えなくなりますが、
デバッグしにくくなったりするのでひとつだけにしておく方が好みです。
PEP 8でもそれがおすすめされています。

```py
class Interpreter:
    def __init__(self, source):
        self._source = source
        self._current_position = 0
```

１文字見るためのメソッドを定義します。
ソースコードの最後まで行ったら`$EOF`という文字列[^eof]を返すことにします。

[^eof]: `char`と言いつつ文字列を返してるじゃないか、と思ったら
`$EOF`という特殊な文字だと思いましょう。

```py
    def _current_char(self):
        if self._current_position < len(self._source):
            return self._source[self._current_position]
        else:
            return "$EOF"
```

インタプリタ本体に入ります。

１文字ずつ読んで、
まずはスペースを読み飛ばし、
次にアルファベットが続くだけ読み進み、
アルファベット以外の文字が出てきたらそこまでの文字列を取り出します。
最後に、取り出したものが`print`かどうかを確認します。

```py
    def interpret(self):
        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        while  self._current_char().isalpha(): self._current_position += 1
        command = self._source[start:self._current_position]
        assert command == "print"
```

ソースコードのおかしいところは`assert`で見つけて即終わらせることにします。
minilangのエラー処理はそれだけです。

普通はできるだけいろいろなエラーを見つけられるよう、できるだけ処理を継続します。
そうすると今度は最初のエラーに引きずられて大量にエラーが出たりするので
そうならないよう手を打つ必要があります。

続いて、数とセミコロンを読み込みます。
処理自体はほとんど同じです。
数は`int`に変換して覚えておくとか、セミコロンは1文字だけ読み進むとか、それくらいです。

```py
        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        while  self._current_char().isnumeric():
            self._current_position += 1
        assert self._source[start:self._current_position].isnumeric()
        number = int(self._source[start:self._current_position])

        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        self._current_position += 1
        assert self._source[start:self._current_position] == ";"
```

セミコロンを読んだら、空白を読み飛ばしてソースの最後までたどりついたか確認します。

```py
        while self._current_char().isspace(): self._current_position += 1
        assert self._current_char() == "$EOF"
```

ソースの最後まで読んで内容の解釈もできたのでいよいよ実行です。

```py
        print(number)
```

ソースの入力を受け取り、実行するループを作ります。
これで全部。

```py
if __name__ == "__main__":
    while src := "\n".join(iter(lambda: input(": "), "")):
        Interpreter(src).interpret()
```

`while`の行は若干詰め込み気味です。

`:=`はウォルラス演算子と呼ばれ、Python 3.8から導入されました。
Python中心な人だとなじみのない人もいるかもしれません。
Pythonの `a = 3` は実は「文」で、値を持たず式のパーツになることはできませんでした。
`a := 3` は3という値を持つ式なので、式の中で組み合わせて使うことができます。
式とか文ってなによ？というひとも、この本を読んでいくと何をいっているのか
わかるようになるかもしれません。

`iter(f, s)`は``f()`が`s`になるまで値を出し続けるイテレータです。
`iter(lambda: input(": "), "")`は`input()`が
空文字列を返す（＝Enterを2回押す）まで入力を1行ずつ返してくれます。

最後に、からっぽでないものは真と判断されるというPythonの性質を使って、
`!= ""`という比較を省いています。
省略してわかりづらくならないかが気になるところですが、
○○が空になるまで繰り返す、と読めばいいのでまあまあ直感的に理解できるかなと。
1行の密度が高まって行数が減り、ひとめで見える情報量が増えていい感じです。

これらを使わないで書くとこうなります（正確には少し違う）。
`while True`＋`if: break`の組み合わせになってしまうのがうれしくないと
思っています。

```py
    while True:
        src = ""
        while True:
            line = input(": ")
            if line == "": break
            src += line + "\n"
        if src == "": break
        Interpreter(src).interpret()
```

## 余談：ウォルラス演算子

Pythonは[^walrus]

[^walrus]: 読みづらく間違いのもとと判断された
ウォルラス演算子の導入
```py
while True:
    a = input()
    if a == "" : break
    ...
```
と書くより
``` py
while (a = input()) != "":
    ...
```
と書きたいという人が増えたんでしょう
`:=`と書くことにしたのは

[PEP 572 – Assignment Expressions | peps.python.org](https://peps.python.org/pep-0572/)
かなりたいへんだったみたい

[PEP 20 – The Zen of Python | peps.python.org](https://peps.python.org/pep-0020/)


ご存じの方も多いと思いますがこの`while`ループは、REPL（Read-Eval-Print-Loop）と呼ばれます。
minilangでは`input()`でReadして、`interpret()`でEvalして、`while`でループはするんですが
minilangのプログラムは値を返さないので値のPrintがなく、厳密にはRELと呼ぶしかないシロモノです。
たいした話ではないですけれども。
なおprint文で出力するのはREPLでいうところのPrintには入りません。

では次行きましょう。