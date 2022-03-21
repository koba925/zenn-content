---
title: "Zenn公式さんにしたがってローカルのZenn執筆環境を整える"
emoji: "✍️"
type: "idea"
topics: ["zenn"]
published: true
---

いくつか記事を投稿したところで記事をローカルで書けることを知ったのでやってみることにしました。そんなに長い記事を書くわけでもなく、ブラウザ上で書くのもそんなに不満はないのでなんとなくです。でもこういうのができると気分がいいです。

Zennのローカル執筆環境についてはいろいろ記事が上がっていますが、公式の記事を頼りに進めます。

https://zenn.dev/zenn

ここからから始めるのが状況に合ってよさそう。

https://zenn.dev/zenn/articles/setup-zenn-github-with-export

結論からいうとすんなりできました。なので手順をこと細かに書くよりも、公式記事からはみ出たようなことを中心に書いていこうと思います。

## Gitリポジトリとの連携

この記事にしたがってGitリポジトリとの連携を設定します。

https://zenn.dev/zenn/articles/connect-to-github

まずはリポジトリの作成。名前は記事のをそのままマネして zenn-content でいきます。どうせ見せるものなのでパブリックにしてみます。

その後連携を設定しますが、ここで

> Zennにログインしたうえで、ダッシュボードのデプロイ管理を開きます。

一瞬、ダッシュボードとやらはどこに？ってなりましたが、アカウントのメニューから「GitHubからのデプロイ」を選んだら表示されました。デプロイ対象ブランチは特に考えず main のままに。別のブランチにすると便利な使い方とかあるかなあ？

![deploy-from-gitlab](/images/preparing-local-zenn-writing-environment/deploy-from-gitlab.png)

指定したブランチへプッシュ（またはプルリクエストをマージ）すると記事が公開されるということでイマドキ風なものを感じます。

これはなんだろう？読んだ人からプルリクエストが来る？

![option-to-show-link](/images/preparing-local-zenn-writing-environment/option-to-show-link.png)

そのようです。

https://zenn.dev/zenn/articles/show-github-link

このままにしてどうなるか見てみます。


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

`npx` ってつけないと実行できないのでは、とか思いつつプレビューを起動します（記事の方にはちゃんと `npx`がついてます）。

```
$ npx zenn preview
👀 Preview: http://localhost:8000
```

Webサーバをlocalhostで動かしてるってことか（なるほど）。これならプレビューとホンモノが確実に一致してくれそう。ブラウザからアクセスしてみたところなんの問題もなく表示されました。

![zenn-cli-preview](/images/preparing-local-zenn-writing-environment/zenn-cli-preview.png)

プレビューのサーバはフォアグラウンドで動いているので続きのコマンドが入力できるよういったんCtrl-Zで止めてバックグラウンドで動かしておきます。`npx zenn preview &` で起動するのがよいかな。

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

https://zenn.dev/settings/export からダウンロードして、`/articles`の内容をコピーしてあげるだけです（本を書いてる人は`/books`も）。さっそくプレビューに記事が反映されてます。

ここでいったんこのままコミットしてプッシュしておきます。

```
$ git add .
$ git commit -m "初期化して既存の記事をインポートする"
$ git branch -M main
$ git push -u origin main
```

連携できてます。

![initial-deployment](/images/preparing-local-zenn-writing-environment/initial-deployment.png)

記事もそのまま残ってました。よかった。

## 記事の新規作成

新規に記事を作るときは、`$ npx zenn new:article` でテンプレートを作ってくれますのでこれを修正して作成します。

ハッシュっぽいファイル名で作成されますが、ファイル名がスラッグになりますし、なにより自分が見てわからなくなるのでわかりやすい名前にしておくのがよさそうです。ファイル名（スラッグ）に使ってよいのは「小文字の半角英数字（a-z0-9）、ハイフン（-）、アンダースコア（_）の12〜50字の組み合わせ」だそうです。

先に `.md` ファイルを作った場合は、ファイル先頭のメタデータ（？）だけほかの記事とかからコピペしてやっても大丈夫そうです。新規に作ったファイルにはこういうのがついてました。

```
---
title: ""
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
```

いったんここでこの記事自身をコミットしてプッシュしてみます。まだ `published: false` のままなので下書きとして反映されるはず・・・されました。

![draft-uploaded](/images/preparing-local-zenn-writing-environment/draft-uploaded.png)

下書きを本サイトで見ても見た目は同じ。当然と言えば当然ですがよくできてるなあ。

## 画像を追加する

うまくいきそうなのでもうちょっとがんばってスクリーンショットを貼り付けたいと思います。

画像の貼り付けにはふたつ方法があるようで、ひとつはプレビューの「画像のアップロード」を使う方法。ドラッグ＆ドロップで画像がアップロードできて `![](https://storage.googleapis.com/zenn-user-upload/xxxxxxxx.png` といった Markdownが取得できるというもの。お手軽感があります。

もうひとつは下記の記事にしたがって、リポジトリ内に画像も入れてしまう方法。ベータ版とのことですが、整理上はこっちの方がよさそうなのでこっちでやってみたい気持ちです。

https://zenn.dev/zenn/articles/deploy-github-images

`/images` ディレクトリに画像を保存して、あとはMarkdownで書いてやればよいようです。`/images`の下に階層を追加してもよいそうなので、記事ごとにディレクトリを切ってやるのがいい気がしました。Markdown は自分で書いてやるしかないのかな。

ここでもう１回コミットしてプッシュ。
本サイトのプレビューで画像が表示されない・・・と思ったら

> /images から始まる画像ファイルはプレビューでは表示されません。

と書いてありました。このへんがベータ版ってことかな。ちゃんと表示されるか確認したいところですが、ローカルのプレビューで表示されてればきっと表示されるに違いないと信じて `published: true` でコミットしてプッシュ。

・・・

公開されました！画像も表示されてます！あとこのボタンがつきました。

![propose-edit](/images/preparing-local-zenn-writing-environment/propose-edit.png)

自分の記事に編集の提案が来る気はあまりしませんが、公式の記事に誤字とかあったときにさっと修正のリクエストがあげられたりすると便利かも。

