---
title: "Zenn公式さんにしたがってローカルのZenn執筆環境を整える"
emoji: "✍️"
type: "idea"
topics: ["zenn"]
published: false
---

いくつか記事を投稿したところで記事をローカルで書けることを知ったのでやってみることにしました。そんなに長い記事を書くわけでもなく、ブラウザ上で書くのもそんなに不満はないのでなんとなくです。でもこういうのができると気分がいいです。

Zennのローカル執筆環境についてはいろいろ記事が上がっていますが、公式の記事を頼りに進めます。

https://zenn.dev/zenn

ここからから始めるのが状況に合ってよさそう。

https://zenn.dev/zenn/articles/setup-zenn-github-with-export

結論からいうとすんなりできました。なので手順をこと細かに書くのはやめて、公式記事からはみでたようなことを書いていきます。

## Gitリポジトリとの連携

この記事にしたがってGitリポジトリとの連携を設定します。

https://zenn.dev/zenn/articles/connect-to-github

まずはリポジトリの作成。名前は記事のをそのままマネして zenn-content でいきます。どうせ見せるものなのでパブリックにしてみます。

その後連携を設定しますが、ここで

> Zennにログインしたうえで、ダッシュボードのデプロイ管理を開きます。

一瞬、ダッシュボードとやらはどこに？ってなりましたが、アカウントのメニューから「GitHubからのデプロイ」を選んだら表示されました。デプロイ対象ブランチは特に考えず main のままに。別のブランチで連携したくなることってあるかなあ？

指定したブランチへプッシュ（またはプルリクエストをマージ）すると記事が公開されるということでイマドキ風なものを感じます。

## Zenn CLIのインストール

次はZenn CLIのインストールです。

https://zenn.dev/zenn/articles/install-zenn-cli

Windows なので Windows 環境でやるか WSL2 でやるかちょっと悩むところ。

エディタは vscode 使うのできっとどちらでやっても問題ないはず。WSL2 上に作るとうっかりディストリビューションごと消しちゃうのが怖いけど GitHub にバックアップあるわけだし、Git とか Node.js とか使うので WSL2 側でやるのが素直かなあ？プレビューがうまく動いてくれるだろうか？どういうしくみで動くんだろう？

決めきれないけど WSL2 で進めました。

まずはさっき作ったからっぽのリポジトリをクローン。

```
$ git clone https://github.com/xxxxxxxx/zenn-content.git
Cloning into 'zenn-content'...
warning: You appear to have cloned an empty repository.
```

ディレクトリに移動してCLIをインストール・初期化（Node.jsはインストール済み）。

```
$ cd zenn-content/
$ npm init --yes
$ npm install zenn-cli
$ npx zenn init

  🎉  Done!
  早速コンテンツを作成しましょう

  👇  新しい記事を作成する
  $ zenn new:article

  👇  新しい本を作成する
  $ zenn new:book

  👇  投稿をプレビューする
  $ zenn preview
```

`npx` ってつけないと実行できないのでは、とか思いつつプレビュー（記事の方にはちゃんと `npx`がついてます）。

```
$ npx zenn preview
👀 Preview: http://localhost:8000
```

Webサーバーをlocalhostで動かしてるってことかーなるほど。これならプレビューとホンモノが確実に一致してくれそう。ブラウザからアクセスしてみたところなんの問題もなく表示。

フォアグラウンドで動いているのでいったんCtrl-Zで止めてバックグラウンドで動かしておきます。`npx zenn preview &` で起動するのがよいかな。

```
$ npx zenn preview
👀 Preview: http://localhost:8000
^Z
[1]+  Stopped                 npx zenn preview
$ bg
[1]+ npx zenn preview &
```

## 既存記事の移行

ここでもとの記事に戻って既存の記事を移行します。

https://zenn.dev/settings/export からダウンロードして、それっぽいフォルダにコピーしてあげるだけです。さっそくプレビューに記事が反映されてます。

ここでいったんこのままコミットしてプッシュしておきます。コマンドラインでやるとこうかな。

```
$ git add .
$ git commit -m "初期化して既存の記事をインポートする"
$ git branch -M main
$ git push -u origin main
```

できてます。
記事を見ても大丈夫そう。

## 記事の新規作成

新規に記事を作るときは、`$ npx zenn new:article` でテンプレートを作ってくれますのでこれを修正して作成します。

ハッシュっぽいファイル名で作成されますが、ファイル名がスラッグになりますし、なにより自分が見てわからなくなるのでわかりやすい名前にしておくのがよさそう。ファイル名（スラッグ）に使ってよいのは「小文字の半角英数字（a-z0-9）、ハイフン（-）、アンダースコア（_）の12〜50字の組み合わせ」だそうです。

先に `.md` ファイルを作った場合は、ファイル先頭のメタデータ（？）だけほかの記事とかからコピペしてやれば大丈夫そうです。新規に作ったファイルにはこういうのがついてました。

```
---
title: ""
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
```

いったんここでこの記事自身をコミットしてプッシュしてみます。まだ `published: false` のままなので下書きとして反映されるはず。