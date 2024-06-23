---
title: "字句解析・構文解析・評価"
---

前章ではprint文だけの言語を作成しました。
これから、少しずつ言語を拡張していきます。
今後の拡張がやりやすいように前回のコードを整理します。

* コードからひとつずつ単語を取り出す - 字句解析
* 単語のつながりを文や式として認識する - 構文解析
* 構文解析の結果を読んで実行する - 評価

この本では基本的にはすこしずつ既存のコードに追加していく、という形で進めていきますが、
ここではほぼ全書き換えになります。
それでも、前回のコードのおもかげを感じることはできるでしょう。

## 字句解析

入力ソースコードはただの文字列（文字が並んだだけのもの）なのでなにも構造がありません。
ここから単語を切り出して、何の単語か決めることを字句解析（Scanning）、
字句解析するところ（字句解析器）をスキャナ(Scanner)と呼びます。
また、切り出された単語をトークン（Token）と言います。

前章のコードでは、以下のような処理が字句解析にあたります。

```py
        while self._current_char().isspace(): self._current_position += 1
        start = self._current_position
        while  self._current_char().isalpha(): self._current_position += 1
        command = self._source[start:self._current_position]
```

普通はTokenクラスを作って、何の種類か、入力ソースコードのどの位置にあるか、
どんな値かなどを覚えさせておくのですが、minilangではPythonの値をそのまま使っています。

* `print`のような名前はPythonの文字列
* `;`のような記号は１文字だけの文字列[^1]
* 数はPythonの数値

[^1]: minilangでは記号は必ず1文字でトークンになるように決めています。
世の中の言語は`==`などたいてい2文字以上の記号のトークンを持っています。

という形で区別をつけています。

今まで「名前」と言っていましたが、業界用語では識別子（Identifier）と言います。
識別子は英文字のあとに、英文字または数字またはアンダースコアが続いたもの、と決めます。

では`Scanner`クラスを作ります。

`Scanner`クラスの仕事は次のトークンを返すことです。


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
```

`Scanner`がトークンを切り出してくれると残りはすっきりします。

```py
class Interpreter:
    def __init__(self, source):
        self.scanner = Scanner(source)

    def interpret(self):
        command = self.scanner.next_token()
        assert command == "print", f"`print` expected, found `{command}`."

        number = self.scanner.next_token()
        assert isinstance(number, int), f"Number expected, found `{number}`."

        semicolon = self.scanner.next_token()
        assert semicolon == ";", f"Semicolon expected, found `{semicolon}`."

        eof = self.scanner.next_token()
        assert eof == "$EOF", f"EOF expected, found `{eof}`."

        print(number)

if __name__ == "__main__":
    while src := "\n".join(iter(lambda: input(": "), "")):
        try: Interpreter(src).interpret()
        except AssertionError as e: print(e)
```

## 構文解析と評価

実は`Scanner`クラスはこれで完成で今後手を入れることはありません。
構文解析
抽象構文木(AST: Abstract Syntax Tree)
データ構造と言ってもminilangではただの配列を使います。
普通はやっぱりクラスを作ります。

配列でつくっておくと、デバッグの時にただ`print`してやるだけで見やすく表示されるという利点があったりとか。

`Interpreter`クラスを`Parser`クラスと`Evaluator`クラスに分割します。

replではASTもprintするようにしておきます[^rppel]。
どう構文解析されたかがわかって学習にはとても便利です。

[^rppel]: Read-Parse-Print-Eval-Loopになりました。

```py
class Parser:
    def __init__(self, source):
        self.scanner = Scanner(source)
        self.current_token = ""

    def parse(self):
        command = self._next_token()
        assert command == "print", f"`print` expected, found `{command}`."

        number = self._next_token()
        assert isinstance(number, int), f"Number expected, found `{number}`."

        self._next_token()
        self._consume(";")

        self._consume("$EOF")

        return ["print", number]

    def _consume(self, expected_token):
        assert self.current_token == expected_token, \
               f"Expected `{expected_token}`, found `{self.current_token}`."
        self._next_token()

    def _next_token(self):
        self.current_token = self.scanner.next_token()
        return self.current_token

class Evaluator:
    def evaluate(self, ast):
        match ast:
            case ["print", val]: print(val)
            case _: assert False, "Internal Error."

if __name__ == "__main__":
    evaluator = Evaluator()
    while source := "\n".join(iter(lambda: input(": "), "")):
        try: evaluator.evaluate(Parser(source).parse())
        except AssertionError as e: print(e)

if __name__ == "__main__":
    evaluator = Evaluator()
    while source := "\n".join(iter(lambda: input(": "), "")):
        try:
            ast = Parser(source).parse_program()
            print(ast)
            evaluator.eval_statement(ast)
        except AssertionError as e: print(e)
```

これで大きな骨組みはできました。
