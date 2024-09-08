---
title: "字句解析・構文解析・評価"
---

前章ではprint文だけの言語を作成しました。
これから、少しずつ言語を拡張していきます。
今後の拡張がやりやすいように前回のコードを整理します。

インタプリタの実装では、以下の三つが基本的要素となります。

* コードからひとつずつ単語を取り出す - 字句解析
* 単語のつながりを文や式として認識する - 構文解析
* 構文解析の結果を読んで実行する - 評価

この本では基本的にはすこしずつ既存のコードに追加していく、という形で進めていきますが、ここではほぼ全書き換えになります。
といっても、細部の処理は前回のコードと同じことをしており、

## 字句解析とは

入力ソースコードはただの文字列（文字が並んだだけのもの）なのでなにも構造がありません。
ここから単語を切り出して、何の単語か決めることを字句解析（Scanning）、字句解析するところ（字句解析器）をスキャナ(Scanner)、レキサ（Lexer）などと呼びます。

また、切り出された単語をトークン（Token）と言います。
トークンには、値（`123`や`true`）やキーワード（`print`）、名前（`var_a`）、記号（`+`、`;`）などがあります。
そのため、字句解析器はトークナイザ（Tokenizer）とも呼ばれます。

前章のコードでは、以下のような処理が字句解析にあたります。
空白を読み飛ばした後、切り出されたトークンを`command`に代入しています。
これにより、字句解析以降はもとのソースコードがどうだったか気にせず、print文が書いてあるんだな、と思って処理することができます。

```py
        while current_position < len(source) and source[current_position].isspace():
            current_position += 1

        start = current_position
        while  current_position < len(source) and source[current_position].isalpha():
            current_position += 1
        command = source[start:current_position]
```

普通はTokenクラスを作って、何の種類か、入力ソースコードのどの位置にあるか、どんな値かなどを覚えさせておくのですが、minilangでは以下のようにPythonの値をそのまま使います。

* 数はPythonの数値
* キーワード、名前は文字列
* 記号は１文字だけの文字列[^1]

[^1]: minilangでは記号は必ず1文字でトークンになるように決めています。
世の中の言語は`==`などたいてい2文字以上の記号のトークンを持っています。

minilangの字句解析器は、入力ソースコードの中に`"123"`という文字列を見つけたら`123`という数値を返す、`"print"`や`"if"`を見つけたら`"print"`や`"if"`という文字列を返す、ということをしているというわけです。
これくらいでもなんとか区別がついて最低限プログラムを実行することはできます。

今まで「値」「名前」と言っていましたが、業界用語ではリテラル（Literal）、識別子（Identifier）と呼びます。
リテラルと値は本当は意味が違って、数値で言えばリテラルは数値を表す文字列（入力ソースコードの中に存在するもの）、値はそのリテラルに対応する実際の値のことを言います。
本書でも今後はリテラル・識別子という言葉を使います。

## 字句解析器の実装

字句解析器を独立したクラスにしたソースは以下のようになります。
あとで説明しますので、全体的な雰囲気だけ見てください。
まだ書き直しになる部分もありますが、写経始めるならこの節からがよいかと。

```py
class Scanner:
    def __init__(self, source) -> None:
        self._source = source
        self._current_position = 0

    def next_token(self):
        while self._current_char().isspace(): self._current_position += 1

        start = self._current_position
        match self._current_char():
            case "$EOF": return "$EOF"
            case c if c.isalpha():
                while self._current_char().isalnum() or self._current_char() == "_":
                    self._current_position += 1
                return self._source[start:self._current_position]
            case c if c.isnumeric():
                while self._current_char().isnumeric():
                    self._current_position += 1
                return int(self._source[start:self._current_position])
            case _:
                self._current_position += 1
                return self._source[start:self._current_position]

    def _current_char(self):
        if self._current_position < len(self._source):
            return self._source[self._current_position]
        else:
            return "$EOF"

def interpret(source):
    scanner = Scanner(source)

    while (command := scanner.next_token()) != "$EOF":
        assert command == "print", f"Expected `print`, found `{command}`."

        number = scanner.next_token()
        assert isinstance(number, int), f"Expected number , found `{number}`."

        semicolon = scanner.next_token()
        assert semicolon == ";", f"Expected semicolon, found `{semicolon}`."

        print(number)

import sys

while True:
    print("Input source and enter Ctrl+D:")
    if (source := sys.stdin.read()) == "": break

    print("Output:")
    try: interpret(source)
    except AssertionError as e: print("Error: ", e)
```

ではパーツごとに見ていきます。
`Scanner`クラスでは入力ソースコードと、今読んでいる位置を覚えておきます。

```py
class Scanner:
    def __init__(self, source) -> None:
        self._source = source
        self._current_position = 0
```

`_source`や`_current_position`の前についているアンダースコアは、この属性はクラスの中だけで使いますよ、他からは使わないでね、というしるしです[^underscore]。
メソッドでもクラス内部から呼ばれることだけ想定しているときはアンダースコアをつけます。

[^underscore]:ただのしるしなので、使えば使えてしまいます。
アンダースコアをふたつ並べれば（簡単には）使えなくなりますが、デバッグしにくくなったりするのでひとつだけにしておく方が好みです。
PEP 8でもそれがおすすめされています。

１文字読むための補助メソッドを先に説明します。
ソースコードの最後まで行ったら`"$EOF"`という文字列[^eof]を返すことにします。

[^eof]: `char`と言いつつ文字列を返してるじゃないか、と思ったら`$EOF`という特殊な文字だと思いましょう。

```py
    def _current_char(self):
        if self._current_position < len(self._source):
            return self._source[self._current_position]
        else:
            return "$EOF"
```

ではScannerクラスの主役である`next_token()`メソッドを作ります。
呼ぶと次のトークンを返すメソッドです。
Scannerクラスで先頭にアンダースコアがついてないのは`next_token()`だけです。
これにより`Scanner`クラスの仕事は求められたら次のトークンを返すことですよ、ということを伝えています。

まず空白を読み飛ばします。

```py
    def next_token(self):
        while self._current_char().isspace(): self._current_position += 1
```

`start`に現在位置＝トークンの先頭を覚えたら、今見ている文字（`_current_char()`）によって処理を分けます。

```py
        start = self._current_position
        match self._current_char():
```

match文はPython 3.10から導入されたもので、ちょっと昔にPythonを覚えて最近はやってない、という人だと知らないかもしれません。
match文は値がどのようなパターンにマッチするかで処理を場合分けするものです。
ものすごくざっくり言うとswitch文の超強力版という感じで、minilangでは多用しています。
いろんなパターンが指定できますが、minilangでは一部しか使っていません。
使うパターンについては都度説明していきます。

```py
            case "$EOF": return "$EOF"
```

matchはcaseを上から順に見ていき、合致するパターンがあればその節が実行されます。

ここに出てきたのは一番単純なパターンで、`_current_char()`が`"$EOF"`であれば`return "$EOF"`を実行します。
ソースを最後まで読んだら`next_token()`も`"$EOF"`を返す、ということです。

```py
            case c if c.isalpha():
                while self._current_char().isalnum() or self._current_char() == "_":
                    self._current_position += 1
                return self._source[start:self._current_position]
```

これはキーワードや識別子のトークンを切り出すところです。
識別子は英文字のあとに、英文字または数字またはアンダースコアが続いたもの、とします。

`case c if c.isalpha()`は前半・後半に分かれます。
`case c`は何にでも一致し、cには一致したものが入ります。
`if c.isalpha()`は、`c`がアルファベットであれば一致したことになります。
要するに、今読んでいる文字がアルファベットであれば一致するということです。

whileループで英数字またはアンダースコアが続く間文字を読み進め、returnで切り出した文字列＝トークンを返します。
アルファベットで始まり、英数字またはアンダースコアが0個以上続いたものを返す、と言うと正規表現っぽさが残ってる感じがしますね。

```py
            case c if c.isnumeric():
                while self._current_char().isnumeric():
                    self._current_position += 1
                return int(self._source[start:self._current_position])
```

数字が続いたものを数とします。
ほとんど同じですが、`int`にして返します。

```py
            case _:
                self._current_position += 1
                return self._source[start:self._current_position]
```

キーワードでも識別子でも数でもなければ１文字のトークンとみなして１文字だけ返します。
普通は、許されてない文字だったらエラーにしますが、minilangではそのあたりは省略しています。

`case _:`のアンダースコアはさきほどの`c`と同じく変数です。
今回は`if`もついてないので、なんにでもマッチします。
C系の言語のswitch文の`default`と同じように働きます。

`Scanner`クラスがトークンを切り出してくれると`interpret()`はすっきり見通しがよくなります。

```py
def interpret(source):
    scanner = Scanner(source)

    while (command := scanner.next_token()) != "$EOF":
        assert command == "print", f"Expected `print`, found `{command}`."

        number = scanner.next_token()
        assert isinstance(number, int), f"Expected number , found `{number}`."

        semicolon = scanner.next_token()
        assert semicolon == ";", f"Expected semicolon, found `{semicolon}`."

        print(number)
```

`while (command := scanner.next_token()) != "$EOF":`で代入演算子を使っています。
`next_token()`の返り値を`command`に代入してから`"$EOF"`かどうか比較しています。
入力ソースコードの最後まで読んだら`next_token()`が`"$EOF"`を返してくることを思い出してください。

REPL部分は前節と同じです。

```txt
Input source and enter Ctrl+D:
print 123;
  print  12345
;
Output:
123
12345
Input source and enter Ctrl+D:
```

では次行きましょう。

GitHub: [コード](https://github.com/koba925/minilang-book/blob/2010_scanner) [差分](https://github.com/koba925/minilang-book/compare/1020_my_implementation...2010_scanner)

## 構文解析と評価

すっきりした`interpret()`をさらに構文解析器（パーザ[^pa-za] Parser）と評価器（Evaluator[^evaluator]）に分けます。
今時点ではちょっとやりすぎ感がありますが今後のため。

[^pa-za]: パーサ・パーサー・パーザ・パーザーといろんな読み方がされます。私はパーザ教です。

[^evaluator]: エバリュエータと読む人は見かけません。

字句解析器が`p`と`r`と`i`と`n`と`t`が並んでいたら"print"だ、などというところまでは判断してくれるようになりましたが、まだ`print`と`123`と`;`が順番に並んでいるだけで、これがprint文であるということはわかっていません。
`print`と`123`と`;`が順番に並んでいたら`123`を出力するprint文なんだなと判断するのが構文解析です。
判断したら抽象構文木（AST: Abstract Syntax Tree）を出力します。

ASTはクラスを使って実装することが多いのですが、minilangではただの配列[*list]を使います。
配列であればひとつひとつの要素を定義しなくて済むことと、デバッグの時にただ`print`（これはPythonのprint）してやるだけでコンパクトにそこそこ見やすく表示されるという利点があり全体的にコンパクトになること、および言語処理系の構造的なところには影響せず今後より本格的な処理系を書くことになっても考え方は変わらないことから採用しました。

[*list]: Pythonでは配列のことを「リスト」(list)と言いますが、「リスト」は複数の意味で使われていて混乱を招きそうなので本書では「配列」(array)と呼びます。

↓↓↓ ここから 不要か？移動する？ ↓↓↓
* <プログラム>は<文>が続いたもの
* <文>は`print` <式> `;`という形をしたもの
* <式>は<数>という形をしたもの

print文のASTは`["print", 123]`という形をしていると書きました。
「<プログラム>は<文>が続いたもの」と前章で書いたのを覚えているでしょうか？
<プログラム>のASTはこの形に対応して`["program", <文>, <文>, ...]`とします。
<文>がひとつもない`["program"]`という形も認めることにします。
↑↑↑ ここまで ↑↑↑

`print 123; print 456;`というminilangのソースコードには`["program", ["print", 123], ["print", 456]]`というASTを対応させます。

## 構文解析器

構文解析を行う`Parser`クラスから。

```py
class Parser:
    def __init__(self, source):
        self.scanner = Scanner(source)
        self._current_token = ""
        self._next_token()
```

入力ソースコードを受け取って字句解析に渡し、`_current_token`に先頭のトークンを読み込んでいます。
トークンの読み込みは`_next_token()`で行います。

```py
    def _next_token(self):
        self._current_token = self.scanner.next_token()
        return self._current_token
```
`_next_token()`は次のトークンを読んで`_current_token`に保存し、返しています。

入力ソースコードを最後まで読むと`Scanner.next_token()`は`"$EOF"`を返してきます。
`_next_token()`もそのまま`"$EOF"`を返します。

次は構文解析の本体を見ましょう。
`parse_program()`は`Parser`クラス唯一のパブリックメソッドです。

```py
    def parse_program(self):
        program: list = ["program"]
        while self._current_token != "$EOF":
            self._check_token("print")
            number = self._next_token()
            assert isinstance(number, int), f"Expected number , found `{number}`."
            self._next_token()
            self._consume_token(";")
            program.append(["print", number])
        return program
```

`["program", ["print", 123], ["print", 456]]`といったASTを返してくれるのがわかるでしょうか？

`program: list = ...`の`: list`は型ヒントです。
本書のPythonコードには基本的には型ヒントはつけていませんが、ここは何かしら書いておかないと警告が出てしまったので最低限の型ヒントをつけています。
今後もときどき出てきますが、このパターンだけなので型ヒントを知らなくても大丈夫です。

`_consume_token(";")`という補助メソッドを呼んでいます。
これは「次はセミコロンが来るはずだから確認だけして捨てる」というものです。
同じことをあちこちでするので補助メソッドにしています。
確認だけして捨てないということもあるのでさらに確認だけを行う処理を`_check_token()`に分離しています。

```py
    def _consume_token(self, expected_token):
        self._check_token(expected_token)
        return self._next_token()

    def _check_token(self, expected_token):
        assert self._current_token == expected_token, \
               f"Expected `{expected_token}`, found `{self._current_token}`."
```

これで構文解析器ができました。

## 評価

構文が解析できると、このプログラムが何をしようとしているのかがわかったことになります。
あとはそれを評価してやるだけです！

`Evaluator`クラスを作ります。

### <プログラム>の評価

`Evaluator`クラスのパブリックメソッドは`eval_program()`です。
<プログラム>のASTを受け取り、評価します。
minilangのプログラムは1個の<プログラム>で、その中に複数の<文>を含んでいるのでした。
`["program", <文>, <文>, ...]`のような形です。
先頭が`"program"`であることを確認し、<文>をひとつずつ評価します。
<文>の評価は`_eval_statement()`に任せます。

```py
class Evaluator:
    def eval_program(self, program):
        match program:
            case ["program", *statements]:
                for statement in statements:
                    self._eval_statement(statement)
            case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

`case ["program", *statements]:`というパターンが出てきました。
こういうのがmatchの便利なところで、先頭が`"program"`であるリスト、つまり`["program", ...]`に一致します。
`"program"`に続く0個以上の要素がリストとして`statements`に取り出されます。
`program`が`["program", ["print", 123], ["print", 456]]`であれば、`statements`は`[["print", 123], ["print", 456]]`となります。

パターンに合致することの判定と、含まれる要素を取り出すことが同時に行えます。
このあたりがmatch文のうれしいところです。

次のパターンは`case unexpected:`です。
これはさきほど出てきた`case _:`とほぼ同じでなんにでもマッチしますが、`unexpected`に値が入ります。

`Parser`には人間が書いたソースコードに影響を受けて知らないパターンが入力されることがありますが、そういったものは`Parser`でエラーになっているはずですので`Evaluator`にはおかしな形の入力は入ってこないはずです。
入ってきたらminilang自体のバグということになります。
そのため、マッチする変数を`unexpected`という名前にし、エラーメッセージも"Internal error"としています。

### <文>の評価

`_eval_statement()`でひとつの<文>を評価します。
`<文>`のASTは`["print", 123]`のような形です。

```py
    def _eval_statement(self, statement):
        match statement:
            case ["print", val]: print(val)
            case unexpected: assert False, f"Internal Error at `{unexpected}`."
```

`statement`が`["print", val]`というパターンにマッチしたら、`val`を出力し、マッチしなければInternal Errorにします。

## REPL

REPLを修正します。
以下のコードはdiffの形で表示しています[^pseudo-diff]。
行頭に`+`がついているのが新しくj追加した行で、`-`がついているのが以前のコードから削除された行です。
以下の例では`Evaluator()`クラスを作っている行が追加され、try～exceptのコードが修正されているということを示します。

[^pseudo-diff]: なお、本書のdiffは実際にdiffを取った結果そのものではありません。
変更箇所をわかりやすく示すためのdiff「風」の書き方と思ってください。
インデントしか変わってないところは変わっていても変更箇所に入れなかったりしています。

```diff py
 import sys
 
+evaluator = Evaluator()
 while True:
     print("Input source and enter Ctrl+D:")
     if (source := sys.stdin.read()) == "": break
 
     print("Output:")
-    try: interpret(source)
-    except AssertionError as e: print("Error: ", e)
+    try:
+        ast = Parser(source).parse_program()
+        print(ast)
+        evaluator.eval_program(ast)
+    except AssertionError as e:
+        print("Error:", e)
```

`Evaluator`や`Parser`を作ってメソッドを呼ぶようにするのが主な修正箇所です。
`Parser`は入力・評価のたびに作り直していますが、`Evaluator`は１回作ったらあとは継続して使いまわしています。
これは、`Evaluator`には定義した変数の情報を記録する予定なので、実行のたびに消えてしまわないようにするためです。

また、ASTもprintするようにしています[^rppel]。
どう構文解析されたかがわかってデバッグや学習にとても便利です。

[^rppel]: Read-Parse-Print-Eval-Loopになりました。

これでこのバージョンのminilangは完成です。

### 実行

では実行します。

```
$ python minilang.py 
Input source and enter Ctrl+D:
print 1;
Output:
['program', ['print', 1]]
1
Input source and enter Ctrl+D:
   print   123  ;
print
  456;
Output:
['program', ['print', 123], ['print', 456]]
123
456
Input source and enter Ctrl+D:
```

`print 1;`という入力ソースコードが`['program', ['print', 1]]`というASTに変換され、`1`が出力されました。
成功です。

これで字句解析器、構文解析器、評価器という大きな骨組みができました。
この骨組みは今後も変わりません。

GitHub: [コード](https://github.com/koba925/minilang-book/blob/2020_parser_evaluator) [差分](https://github.com/koba925/minilang-book/compare/2010_scanner...2020_parser_evaluator)

## 余談：クラス・関数・メソッドの並べ方

クラスや関数の並べ方にはいくつか観点があります。


クラスの中では、パブリックなメソッドを上に、それ以外を下

これは逆にされることはあまりないように思います。
C系の言語ではクラス定義

使う側を上に、使われる側を下に、またはその逆

ひとつにはどちらが読みやすいかという観点がありますが、言語処理系の都合もあります。
というのは、定義が終わっていないとそれを使うことができない処理系では基本的に呼び出される側を先に書くことになります。
まったくそれしかできないと相互に呼び出しあう関数が書けなくなってしまうのでなにかしら逆でも

Pythonでは、ソースコード上は


トップダウン的に並べる流儀とボトムアップ的に並べる流儀があるようです。





呼び出し順
呼び出される方が上 呼ぶ方が上

アルファベット順
