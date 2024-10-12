---
title: "minilangで二分探索木を作ってみる"
emoji: "🔍"
type: "tech"
topics: ["computerscience", "algorithm", "minilang"]
published: false
---

本記事は、「●」（以下「minilang本」）の関連記事です。

minilangは数とか真偽値とか、単独の値しかなくて配列とか辞書とかのデータ構造を持ってないのがすこし残念なところです。
といってもデータ構造をまったく扱えないわけではないんです。クロージャがあるので。

二分探索木を作ってみましょう。本筋ではないので説明は最低限です。

まず二分探索木のノードを作ります。`make_new`で`val`をくれと言われたら`val`の値を返す関数を作って返します。`node_val`はその関数に`val`をくれとお願いして`val`の値を取得します。クロージャが`val`の値を覚えてくれているので、これでノードに値を記憶しておくことができます。

101とか102とかの値には意味はありません。区別したいだけです。文字列があればいらなかったんですけど。あるいは101とかの数字をそのまま使うとか（嫌

```
Input source and enter Ctrl+D:
var method_val = 101;
var method_left = 102;
var method_right = 103;

def node_new(val, left, right) {
    return func(method) {
        if method = method_val { return val; }
        if method = method_left { return left; }
        if method = method_right { return right; }
    };
}

def node_val(this) { return this(method_val); }
def node_left(this) { return this(method_left); }
def node_right(this) { return this(method_right); }
```

なんとなく`node`オブジェクトを定義してる感じがしませんか？命名もそのへんを意識しています（`new`とか`method`とか`this`とか）。

ノードができてしまえばあとは普通に書けます。二分探索木に値を追加したり、全体を見て回ったり、値を見つけたりする関数を作ります。`None`や`null`がないので`false`で代用します。

```
Input source and enter Ctrl+D:
def bst_put(bst, val) {
    if bst = false { return node_new(val, false, false); }

    var cur_val = node_val(bst);
    if cur_val = val {
        return bst;
    } elif less(val, cur_val) {
        return node_new(cur_val, bst_put(node_left(bst), val), node_right(bst));
    } else {
        return node_new(cur_val, node_left(bst), bst_put(node_right(bst), val));
    }
}

def bst_walk(bst) {
    if bst = false { return; }

    bst_walk(node_left(bst));
    print node_val(bst);
    bst_walk(node_right(bst));
}

def bst_find(bst, val) {
    if bst = false { return false; } 
    
    var cur_val = node_val(bst);
    if cur_val = val {
        return val;
    } elif less(val, cur_val) {
        return bst_find(node_left(bst), val);
    } else {
        return bst_find(node_right(bst), val);
    }
}
```

では実際に二分探索木を構築します。

```
Input source and enter Ctrl+D:
var bst = false;
set bst = bst_put(bst, 7);
set bst = bst_put(bst, 3);
set bst = bst_put(bst, 1);
set bst = bst_put(bst, 9);
set bst = bst_put(bst, 5);
```

二分探索木を見て回ります。昇順になっています。

```
Input source and enter Ctrl+D:
bst_walk(bst);
Output:
1
3
5
7
9
```

0から10をひとつずつ検索します。二分探索木に含まれない数のところでは`false`が出力されます。

```
Input source and enter Ctrl+D:
var i = 0;
while less(i, 10) {
    print bst_find(bst, i);
    set i = i + 1;
}
Output:
false
1
false
3
false
5
false
7
false
9
```

できました！