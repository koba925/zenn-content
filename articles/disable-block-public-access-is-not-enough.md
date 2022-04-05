---
title: "AWS S3でBlock Public Accessを無効にするだけではパブリックアクセスにならない"
emoji: "😲"
type: "tech"
topics: ["aws", "s3"]
published: true
---

とある AWS 入門にしたがって WordPress サイトを構築し、Offload Media Liteというプラグインで画像を S3 から配信するように設定したのですがうまくいきません。

調べて解決したのでメモを作成しました。

## 起こったこと

ざっくり言うとこんな感じでWordPressとOffload Media Liteをインストール・設定したのですが、画像を投稿してもS3からの配信になりませんでした。

1. EC2インスタンスを起動し、WordPressをインストール。画像はEC2から配信。
1. AmazonS3FullAccessを持つIAMユーザを作成。
1. S3のBucketを作成して、Block all public accessを解除。
1. WordPressの管理画面からOffload Media Liteのプラグインをインストールして有効化。
1. Offload Media Liteにアクセスキーや使用するBucketを設定。

Offload Mediaの[Amazon S3 Quick Start Guide](https://deliciousbrains.com/wp-offload-media/doc/amazon-s3-quick-start-guide/#offload-existing)を見ても大丈夫そうに見えるのですが・・・

なおEC2で動かしているのでIAMユーザのアクセスキーではなくIAM Roleでやる方がスジだと思いますがまだ試していません。

## 結論

S3のObjectをパブリックに公開するには以下のふたつを実施する必要があります。

* Block Public Accessを無効にする
* ACLでオブジェクトに対するEveryoneからのReadを許可する

ところが、Bucketを作った時の初期値ではACLがdisableになっているため、enableにしてやる必要があります。ACLのenable化はBucketのObject Ownership設定で行います。

なお、AWSはACLはdisableでBlock public accessも有効のままにしておくことを推奨しています。そもそもS3をパブリックアクセスにすることをおすすめしてないということですね。

https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/acls.html

https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/access-control-block-public-access.html

## 検証

まずはOffload Media Liteは使わず、Management ConsoleでS3を使って確かめます。

1. Bucketを作成し、画像ファイルをアップロード → アクセス不可

まずはデフォルトの設定のまま、ブラウザでObject URLにアクセスしました。当然アクセスできません。こんなのが表示されてAccess Deniedになります。

```
This XML file does not appear to have any style information associated with it. The document tree is shown below.
<Error>
<Code>AccessDenied</Code>
<Message>Access Denied</Message>
<RequestId>TKFGG78HX4MW4R88</RequestId>
<HostId>F9BeSexfHqwhNMpCp7KyJTWPv8K2aSIe/7GWl/srakSHXz/O5kZlE0StSd8ua2+HnU0XSqYZnyk=</HostId>
</Error>
```

2. BucketでBlock all public accessを解除 → アクセス不可

ここまではすでにやったところになります。Permissions overviewにObjects can be publicと表示されます。

まだオブジェクトにはアクセスできません。"Can be"というのがミソですね。

3. ACLをenable化

オブジェクトにACLを設定してやりたいのですが、その前にまずACLをenableする必要があります。

まず、BucketのPermissionsタブでObject OwnershipのEditボタンをクリックします。「ACLs disabled (recommended)」と「ACLs enabled」から選択する画面になりますので、「ACLs enabled」を選択し、確認のチェックボックスをチェックしてからSave Changesボタンをクリックします。

まだオブジェクトにはアクセスできません。

4. オブジェクトのACLでEveryoneにReadを許可

公開したいオブジェクトを選択し、PermissionsのタブでAccess control list (ACL)のEditボタンをクリックします。Edit access control listの画面になりますので、Everyone (public access) のObjectでReadを有効にしてやります。さきほどと同様に、確認のチェックボックスをチェックしてからSave Changesボタンをクリックします。よほど気を付けてほしいと見えます。

これで表示されるようになりました！

5. BucketでBlock all public accessを再開

ACLはそのままにして、Block all public accessを有効に戻してみます。Permissions overviewがBucket and objects not publicに戻りました。

ブラウザからのアクセスもAccess Deniedに戻りました。想定通りです。

## Offload Media Lite での確認

BucketのACLをenableにするだけで新規にアップロードしたものはS3に入るようになりました。オブジェクトに対するACL設定はOffload Media Liteがやってくれているようです。既存のオブジェクトをOffloadするのは有償版の機能っぽく、そこは目的ではないのでこれでよしとします。

なお、S3のBucketを自分で作るのではなく、Offload Media Liteの管理画面から作った場合は初めからACLがenabledになっていて特に考えることもなく使えました。S3の勉強ではなく単にWordPressを使いたいだけならそのほうがいいですね。というか、AWSと同じく、Offload Media LiteもS3をPublicにするのではなくCloudFrontから配信するのを推奨していますのでそのほうがいいでしょう。

https://deliciousbrains.com/wp-offload-media/doc/block-all-public-access-to-bucket/

私の見ているAWS入門でも続いてCloudFrontの設定を行っています。

## おわりに

ACLをenableする手順がAWS入門やOffload Media Liteのドキュメントに入ってないのは、昔は不要だったけど後でS3 の Public Access に関するアクセス制御が厳しくなった、ということかなあと想像しています（未確認）。
