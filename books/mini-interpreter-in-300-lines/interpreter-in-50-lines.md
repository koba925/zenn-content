---
title: "150行くらいでインタプリタ"
---

入力が面倒で文法が見慣れなくてコードが見づらいのをがまんすれば！
50行でインタプリタらしきものが書けます。

原理中の原理

文字列をリストに変換するために`eval`を使います[^eval]。

[^eval]: `eval`を許すなら`eval(input())`でPythonインタプリタを作りましたって言えるんじゃね？っていうツッコミはなしで。

```
$ python minilang.py 
Input source and enter Ctrl+D:
['program', 
    ['var', 'fib', ['func', ['n'], ['block', 
        ['if', ['=', 'n', 1], 
            ['block', ['return', 1]],
            ['block']],
        ['if', ['=', 'n', 2],
            ['block', ['return', 1]],
            ['block']],
        ['return', ['+', ['fib', ['-', 'n', 1]], ['fib', ['-', 'n', 2]]]]]]],
    ['print', ['fib', 6]]
]
Output:
8
```

なんとなくプログラミング言語に見えてきましたね？

というか実のところ、最初にできたのはPythonのリストで書いたLisp風コードの評価器なのでした。
それができてから、「これに字句解析と構文解析かぶせたら普通のインタプリタになるんじゃね？」ということでminilang（的なもの）が始まったのでした。
やっていく中で評価器自身もLispからminilang寄りに直していきましたが（`lambda`を`func`にしたりとか）まあだいたい似たようなものです。