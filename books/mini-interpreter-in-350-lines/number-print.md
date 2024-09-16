---
title: "数とprint文だけの言語"
---

最初は、すぐ作れるように最低限の仕様で言語を作ります。
どれくらい最低限かというと、数とprint文だけの言語を作ります。
`print 123;`を実行すれば`123`と表示される、それだけ、という言語です。
数の計算もしません。複数回書けるようにはしておきます。

たとえば

```
print 1; print 12;
print 123;
```

というソースを実行すると

```
1
12
123
```

と表示される、ということです。

すこし細かく文章にしてみます。

* <プログラム>は<文>が続いたもの
* <文>は`print` <式> `;`という形をしたもの
* <式>は<数>という形をしたもの
* <数>は数字が続いたもの
* `print`、<数>、`;`の前後は任意の空白を入れてよい
* print文は<式>の計算結果を表示する

ちゃんとした本ではもっと厳密に定義します[^bnf]が、この本は雰囲気がわかればいいや、あとはコードで、というスタイルで行きます。

[^bnf]: 文法を定義するときは、BNF記法（Backus Naur Form バッカス・ナウア記法）と呼ばれる形式で書くのが普通です。
たとえば、`<program> ::= <statement> | <statement> <program>`のように書きます。
さらに普通は、BNF記法をもう少し人にやさしくしたEBNF記法（Extended BNF Form）を使います。

## 最初の実装

さて、名前がないと説明しづらいのでこの本で作っていく言語を「minilang」と呼ぶことにします。

上に書いた「こういう言語です」をコードにしました。
先におことわりしておくと、本章のコードはあとで書き直しになる部分も多いので、写経しながら読んでいこうという方も読むだけで通過しても大丈夫です。

ファイル名はminilang.pyとしましょう。
GitHubでは https://github.com/koba925/minilang-book/blob/1010_regex_implementation にあります。

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

説明はおいておいて、何はともあれ動かしてみます。
起動方法はお好きなように。
下の例では単純にコマンドラインから起動しています。

`Input source and enter Ctrl+D:`の下でカーソルが入力を待っていますので、上記文法にしたがって、あるいは従わずにminilangのプログラムを入力します。
1行ごとにEnterを押し、入力し終わったらCtrl+Dを押してください。
入力したプログラムが実行されます。
Windows上でそのまま動かしている場合は、Ctrl+Dの代わりにCtrl+Zを押し、さらにEnterを押してください。

最初から何も入力せずにCtrl+Dを押すか、エラーが出ると終了します。

```
$ python minilang.py
Input source and enter Ctrl+D:
print 1; <Enter>
<Ctrl+D>
Output:
1
Input source and enter Ctrl+D:
```

ちゃんと実行されました。

空白や改行を入れても大丈夫です。

```
Input source and enter Ctrl+D:
  print  123  ;  print       <Enter>
12345 <Enter>
   ; <Enter>
<Ctrl+D>
Output:
123
12345
Input source and enter Ctrl+D:
```

入力ソースコードに誤りがあると実行しません。
なお本書では、minilangで書かれたプログラムのソースコードと、minilangを実行するプログラムのソースコードを特に区別したいときは、前者を入力ソースコード、後者を実行ソースコードと呼ぶことにします。

```
Input source and enter Ctrl+D:
prin 1; <Enter>
<Ctrl+D>
Output:
Error:  Does not match.
Input source and enter Ctrl+D:
```

一般的なものとは少し違うところもありますが、こうやってちょっとずつコードを入力してはその場で実行してくれるものをREPLと言います。
読んで（Read）、評価して（Eval）、出力して（Print）、ループする（Loop）の略です[^repl]。

[^repl]:minilangでは`sys.stdin.read()`がR（Read）、`interpret()`がE（Eval）、`while`がL（Loop）にあたります。
P（Print）にあたる部分がないので厳密にはRELということになります。
REPLでいうところのPrintはEvalの結果を出力するものですが、minilangの`interpret()`は値を返さないためです。
print文で文字を出力するのはここでいうPrintにはあたりません。

ではソースコードを見ていきましょう。

```py
def interpret(source):
    statement_pattern = re.compile(r"\s*print\s*(\d+)\s*;\s*")
```

`interpret()`がインタプリタ本体です。
`re.compile`で、`print <数>;`に対応するパターンを作っています。

次が実際に入力ソースコードを読んで実行する部分です。

```py
    current_position = 0
    while current_position < len(source):
        statement = statement_pattern.match(source, current_position)
        assert statement is not None, "Does not match."
        print(int(statement.group(1)))
        current_position = statement.end()
```

パターンにマッチさせては`<数>`の部分を出力し、まだマッチしてない部分の先頭に進む、というのを入力ソースコードの末尾まで繰り返します。

正規表現[^regular-expression]のおかげでとても短く書けました。
正規表現に慣れてない人は「？」となっているかもしれませんが、次に自前で書き直す部分なのでさらっと読み飛ばしても問題ありません。
正規表現っていう言葉、プログラミングでよく出てくるからと何げなく使っている人も多いと思いますが、計算機科学の世界ではよく研究された分野です。
「正規表現」「オートマトン」「形式言語」あたりのことばで検索してみるといろいろ出てきます。

[^regular-expression]: `r"^\s*print\s*(\d+)\s*;\s*$"`といった正規表現はひとつの言語とみなすことができます。PythonやJavascriptなどのようにいろいろなことができる言語ではなく、文字列がある形になっているかを判定するという目的にのみ使われる言語なのでDSL (Domain Specific Language) とも呼ばれます。特定の目的に絞っているのでその目的に対してはうまく書けるようになっています。

次はREPLの部分です。

```py
while True:
    print("Input source and enter Ctrl+D:")
    if (source := sys.stdin.read()) == "": break
```

標準入力から入力ソースコードを受け取っています。
`input()`ではなく`sys.stdin.read()`を使っているのは、複数行の入力を受け付けるためです。

`:=`はウォルラス演算子と呼ばれ、式の中で変数に代入することができます。
Python 3.8から導入されました。
Python中心な人だとなじみのない人もいるかもしれませんが、こう書いているのと同じです。

```py
    source = sys.stdin.read()
    if source == "": break
```

Pythonの `a = 3` は実は「文」で、値を持たず式のパーツになることはできませんでした。
`a := 3` は3という値を持つ式なので、式の中で組み合わせて使うことができます[^assignments]。
式と文についてはあとでもう一度軽く触れます。

[^assignments]: 詳細は[7.2. 代入文](https://docs.python.org/ja/3/reference/simple_stmts.html#assignment-statements)と[6.12. 代入式](https://docs.python.org/ja/3/reference/expressions.html#assignment-expressions)でご確認ください。

`:=`は`==`よりも優先順位が低いので、`(source := sys.stdin.read()) == ""`とカッコに入れる必要があります。
カッコをつけないと、`source := (sys.stdin.read() == "")`という意味となり、入力が空文字列かどうか（つまり`True`または`False`）が`source`に代入されてしまいます。

なにも入力されていなければwhileループを抜けて、REPLを終了します。

つぎはいよいよ実行です。

```py
    print("Output:")
    try: interpret(source)
    except AssertionError as e: print("Error: ", e)
```

入力ソースコードを引数にしてインタプリタ本体を呼んでいるだけです。
`assert`で例外が発生したら拾ってエラーメッセージを出力し、REPLを継続するようにしています。
その他の例外は拾っていないので、発生したらそこで実行を終了します。

minilangでは、この後もassertでエラーをチェックして、その場で処理を終わらせます。
普通はできるだけいろいろなエラーを見つけられるよう、できるだけ処理を継続します。
そうすると今度は最初のエラーに引きずられて大量にエラーが出たりするので、そうならないよう手を打つ必要があります。
エラーの処理やエラーメッセージも言語処理系の大事なところではありますが、minilangでは簡潔さを優先して、最低限のエラーチェックのみ行っています。

## 正規表現を使わずに実装する

正規表現は便利なのですが、自前で処理するように書き換えます。
与えられたソースコードの文字列を１文字ずつ見ながらひとかたまりずつ処理していきます。

`interpret()`をまるごと以下のコードと入れ替えます。
GitHubは https://github.com/koba925/minilang-book/blob/1020_my_implementation/minilang.py です。
また、https://github.com/koba925/minilang-book/compare/1010_regex_implementation...1020_my_implementation でコードの差分が（大半差分ですが）確認できます。
冒頭にも書きましたが、GitHubを見ずに読み進めても大丈夫なように書いていきますのでご安心を。

なお、このコードもあとでまるごと書き換えますので読むだけでもOKですが、類似の書き方をする部分はありますので理解はしておいてください。

```py
def interpret(source):
    current_position = 0

    while current_position < len(source):
        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        while  current_position < len(source) and source[current_position].isalpha():
            current_position += 1
        command = source[start:current_position]
        assert command == "print", f"Expected `print`."

        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        while  current_position < len(source) and source[current_position].isnumeric():
            current_position += 1
        assert source[start:current_position].isnumeric(), f"Expected number."
        number = int(source[start:current_position])

        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        if current_position < len(source): current_position += 1
        assert source[start:current_position] == ";", f"Expected semicolon."

        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        print(number)
```

さきほどのコードとほぼ同様に動作します。
なお、これ以降、minilang REPLの起動(`$ python minilang.py`)や、`<Enter>`・`<Ctrl+D>`は省略します。

```txt
$ python minilang.py 
Input source and enter Ctrl+D:
print 1;
Output:
1
Input source and enter Ctrl+D:
  print   123   ;   
print
12345
;
Output:
123
12345
Input source and enter Ctrl+D:
trinp 1;
Output:
Error:  Expected `print`.
Input source and enter Ctrl+D:

```

ではソースコードをパーツごとに見ていきます。

```py
def interpret(source):
    current_position = 0
```

`current_position`には、入力ソースコードのこれから読もうとしている文字の位置を格納します。


```py
    while current_position < len(source):
```

入力コードの最後を読むまでwhileループを回します。
`<文>`を繰り返し実行することに対応します。

```py
        while current_position < len(source) and source[current_position].isspace():
            current_position += 1
```

ここからがひとつの`<文>`の処理になります。
まず、空白文字を読み飛ばします。
`isspace()`はタブや改行も空白として扱います。
読み飛ばしている間に入力ソースコードの末尾を超えてしまわないようにチェックも行っています。

```py
        start = current_position
        while  current_position < len(source) and source[current_position].isalpha():
            current_position += 1
        command = source[start:current_position]
        assert command == "print", f"Expected `print`."
```

空白文字を読み飛ばしたら次にアルファベットが続くだけ読み進みます。
アルファベット以外の文字が出てきたらループを抜け、そこまでの文字列を取り出します。
文字列を取り出すには、文字列の先頭位置を覚えておき、先頭位置から現在位置（＝アルファベットでない文字）の１文字手前までを取り出す、というやり方をしています。
`source[start:current_position]`と書くと、`start`の位置から`current_position`の一つ手前までが取り出されますので、1を引く必要はありません。
最後に、取り出したものが`print`かどうかを確認します。

```py
        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        while  current_position < len(source) and source[current_position].isnumeric():
            current_position += 1
        assert source[start:current_position].isnumeric(), f"Expected number."
        number = int(source[start:current_position])

        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        if current_position < len(source): current_position += 1
        assert source[start:current_position] == ";", f"Expected semicolon."

        while current_position < len(source) and source[current_position].isspace():
            current_position += 1
```

続いて、数とセミコロンを読み込みます。
処理自体はほとんど同じです。
数は`int`に変換して覚えておくとか、セミコロンは1文字だけ読み進むとか、それくらいです。

ソースの最後まで読んで内容の解釈もできたのでいよいよ実行です。

```py
        print(number)
```

数を出力したら、次のループへ進みます。

では次行きましょう。

## 余談：代入演算子（ウォルラス演算子）

Pythonはもともとは代入できる演算子を持っていませんでした。
`=`は一見演算子に見えますが、実は代入文を示す構文だったのです。

Python以前の言語（特にC系）にも存在した代入式が初期のPythonで取り入れられていなかったのは、それが可読性を下げたり、バグの温床となりかねないと考えられていたからでしょう。
実際、C言語でも`while ((c = getchar()) != EOF) { putchar(n); }`のようなコードは頻繁に用いられ、C言語プログラマは意識もせずに読めますが、初見では読みづらく、書籍などではたいてい注意書きがついています。
上ではさらっと一見演算子に見えると書きましたが、プログラミングに初めて触れる人には`:=`が`+`や`-`などと同じく演算子であるということ自体違和感があるかもしれません。

それがなぜのちに導入されることになったかというと、ひとことで言えば簡潔に書けるから、ということなんですが特にループを書くときに使いたいというのが大きいのではないかと思います。
代入演算子がないと、キーボードから入力を受け取り、入力がなければループを抜けるという処理は以下のように書く必要があります。

```py
while True:
    a = input()
    if a == "" : break
    ...
```

代入演算子を使うとこのように書けます。

``` py
while (a = input()) != "":
    ...
```

行数が減ったのはもちろんですが、ループの先頭を見ただけで終了条件がわかる、breakを使わなくてよい[^break]、という利点もあります。

[^break]: break文は（多少程度はいいいとは言え）やっていることはgotoなので。

個人的にはうれしい代入演算子ですが、[PEP 572 – Assignment Expressions](https://peps.python.org/pep-0572/)の決定が下されるまでには相当の論争があったようです。
[The Zen of Python](https://peps.python.org/pep-0020/) には「やり方がひとつだけあるのが望ましい」とあり、代入するのに文と式の両方でできるというのはこの考え方に反する、という見方もあります[^if-expression]。
最初から代入が式であればそこが問題になることはなかったはず。

[^if-expression]: ifも文と式の両方がありますが、これはPython 1.0のときからそうだったようです（未確認）。

そういった経緯などを知るのも言語処理系の勉強のおもしろい部分だと思います。
