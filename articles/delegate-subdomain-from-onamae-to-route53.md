---
title: "お名前.comからRoute 53にサブドメインを委任する"
emoji: "📛"
type: "tech"
topics: ["aws", "route53", "onamae"]
published: true
---

AWSお勉強中に お名前.com で取得したドメインを Route 53 に設定して使ってみようと思ったんですが、すでに設定済みの項目もあるのでサブドメインだけ Route 53 に委任することにしました。

以下、 お名前.com で example.com を取っていて、example.com は お名前.com 管理のまま、aws.example.com は Route 53 で管理するようにして Elastic IP に割り当て済みの 198.51.100.1 を設定する、という想定で書いていきます。
なお、宗教上の理由によりマネジメントコンソールは English (US) で表示しています。

全体の流れは以下のページにしたがってやっていきます。

https://docs.aws.amazon.com/ja_jp/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html

既存のサブドメインを委任するときはこちらになります。既存のサブドメインの内容を移行する手順になっています。

https://docs.aws.amazon.com/ja_jp/Route53/latest/DeveloperGuide/MigratingSubdomain.html

## ホストゾーンの作成

まず、 DNS 設定を登録するためのホストゾーンを作成します。

1. マネジメントコンソールで Route 53 を開き、左メニューで Hosted zones を選択します。
1. Create hosted zone のボタンをクリックします。
1. Domain name に aws.example.com と入力します。
1. インターネットからアクセスしたいので、 Type は Public hosted zone （のまま）とします。
1. Description や Tag は任意で。
1. Create hosted zone のボタンをクリックします。

ホストゾーンができて、NSレコードやSOAレコードが作成されています。

## IP アドレスの登録

aws.example.com は 198.51.100.1 ですよ、というレコードを作成します。

1. aws.example.com の詳細を表示し、 Create record のボタンをクリックします。
1. Record name には好きな名前を入れればよいのですが、今回は空にしておきます。
1. Record type は IPv4 のアドレスを登録するので A とします。
1. Value に 198.51.100.1 と入力します。
1. TTL は 300、 Routing Policy は Simple routing のままにしておきます。
1. Create record のボタンをクリックします。

aws.example.com の詳細画面に戻ります。 Records タブに A レコードが追加されているはず。

## サブドメインを委任する

これだけでは aws.example.com の IP アドレスを取得しようとすると Route 53 ではなく お名前.com のサーバーに聞きに行ってしまいますので、お名前.com の方に aws.example.com は Route 53 で管理するよ、という設定を入れてやります。

1. aws.example.com の詳細画面の NS レコードを参照して、ネームサーバーの名前（たぶん4つ）をどこかにコピペしておきます。こんな感じです。
```
ns-xxxx.awsdns-xx.com.
ns-xxxx.awsdns-xx.org.
ns-xxxx.awsdns-xx.co.uk.
ns-xxxx.awsdns-xx.net.
```
2. [お名前.com](https://www.onamae.com/) にログインします。
1. ネームサーバーの設定 > ドメインの DNS 設定 を選択します。「DNS設定/転送設定－ドメイン一覧」画面が表示されます。
1. 対象のドメインを選択して「次へ」ボタンをクリックします。「DNS設定/転送設定－機能一覧」画面に進みます。
1. 「DNSレコード設定を利用する」の「設定する」ボタンをクリックします。「DNSレコード設定」画面に進みます。
1. 「入力」の「A/AAAA/CNAME/MX/NS/TXT/SRV/DS/CAAレコード」でさきほどコピペしたネームサーバーをひとつずつ追加していきます。
Type: NS
ホスト名: aws (.example.com)
TTL: 86400 (のまま)
VALUE: コピペしたネームサーバー名（最後のドットはつけない）
1. 下の方に進んで 「DNSレコード設定用ネームサーバー変更確認」 をチェック（されていることを確認）します。
DNS設定を行うには、ドメインを取得したときに設定されているネームサーバーから、DNS設定が可能なネームサーバーに変更する必要があります。別途自分で変更することもできるようですが、チェックを入れておけば勝手にやってくれるようです。
1. 「確認画面へ進む」ボタンをクリックします。
1. 内容を確認したら「設定する」ボタンをクリックします。

お名前.com から 「DNSレコード設定 完了通知」「ネームサーバー情報変更 完了通知」というメールが届きます。

> ネームサーバー変更するをお選びいただいた場合は最大72時間、DNSレコードの設定のみの場合は数時間程度、反映完了までお時間をいただきます。

だそうですので待ちます。

## 確認する

一晩待ちました。

```
$ dig aws.example.com

; <<>> DiG 9.16.1-Ubuntu <<>> aws.example.com
...
...

;; QUESTION SECTION:
;aws.example.com.               IN      A

;; ANSWER SECTION:
aws.example.com.        0       IN      A       198.51.100.1
ns-xxxx.awsdns-xx.com.  0       IN      A       xxx.xxx.xxx.xxx
ns-xxxx.awsdns-xx.com.   0       IN     AAAA    xxxx:xxxx:xxxx:xxxx::1
...
...
```

できたようです。

ネームサーバーが A レコード・ AAAA レコードとして表示されているのはどういうことでしょうか。いろんな聞きかたをしてみました。

```
$ dig aws.example.com +short
198.51.100.1
xxx.xxx.xxx.xxx
xxxx:xxxx:xxxx:xxxx::1
xxx.xxx.xxx.xxx
xxxx:xxxx:xxxx:xxxx::1
xxx.xxx.xxx.xxx
xxxx:xxxx:xxxx:xxxx::1
xxx.xxx.xxx.xxx
xxxx:xxxx:xxxx:xxxx::1
$ dig @ns-xxxx.awsdns-xx.com aws.example.com +short  # Route 53のネームサーバー
198.51.100.1
$ dig @8.8.8.8 aws.example.com +short                # Googleのネームサーバー
198.51.100.1
$ dig @01.dnsv.jp aws.example.com +short             # お名前.comのネームサーバー
$ ssh ec2-user@aws.example.com
[ec2-user@ip-10-0-10-10 ~]$ digaws.example.com +short # EC2のインスタンスから
198.51.100.1
```

```
PS C:\> Resolve-DnsName aws.example.com

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
aws.example.com                                A      227   Answer     198.51.100.1
```

```
C:\>nslookup aws.example.com
サーバー:  UnKnown
Address:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

権限のない回答:
名前:    aws.example.com
Address:  198.51.100.1
```

一番上の `dig` は WSL2 上で実行してるんですけど、なにか特殊なことでもしているのかなあ？

```
$ dig +trace aws.example.com

; <<>> DiG 9.16.1-Ubuntu <<>> +trace aws.example.com
;; global options: +cmd
;; connection timed out; no servers could be reached
```

えっ？

このへんにしておきます。なにかご存じの方いらしたら教えてください。

## サブドメイン委任を解除する

サブドメインの削除もやってみます。

あまりやる人がいないのかやればできてしまうからなのか、「サブドメイン 委任 解除」ではキーワードがよくないのか、それっぽい情報を見つけることができませんでした。おそらく最後から順番に戻していけば大丈夫でしょう。

まずはサブドメインの委任を解除します。

1. 委任するときと同様にして「DNSレコード設定」画面に進みます。
1. 「登録済み」の「A/AAAA/CNAME/MX/NS/TXT/SRV/DS/CAAレコード」でさきほど登録したNSレコードの「削除」にチェックを入れます。「状態」を「無効」にしてもよさそうですが、さっぱりしたいので「削除」にしました。
1. 下の方に進んで 「DNSレコード設定用ネームサーバー変更確認」 をチェック（されていることを確認）します。
1. 「確認画面へ進む」ボタンをクリックします。
1. 内容を確認したら「設定する」ボタンをクリックします。

まだDNS引けます。

## A レコードの削除

まず A レコードを消さないとホストゾーンの削除ができないようです。

1. Hosted zones の画面で、今回作成したホストゾーンをクリックします。
1. 自分で追加した A レコードにチェックを入れます。
1. Delete record のボタンをクリックします。
1. 確認画面が出ますので、Delete record のボタンをクリックします。
1. Record name には好きな名前を入れればよいのですが、今回は空にしておきます。
1. Record type は IPv4 のアドレスを登録するので A とします。
1. Value に 198.51.100.1 と入力します。
1. TTL は 300、 Routing Policy は Simple routing のままにしておきます。
1. Create record のボタンをクリックします。

NS レコード、SOA レコードはそのままでよいようです。

```
$ dig @ns-xxxx.awsdns-xx.com aws.example.com +short
$ dig aws.example.com +short
198.51.100.1
xxx.xxx.xxx.xxx
xxxx:xxxx:xxxx:xxxx::1
xxx.xxx.xxx.xxx
・・・
```

もうRoute 53のネームサーバーからは消えましたが、世界からは消えていません。

## ホストゾーンの削除

1. Hosted zones の画面に戻り、今回作成したホストゾーンを選択します。
1. Delete のボタンをクリックします。
1. 本当に消すかの画面になるので delete と入力して Delete のボタンをクリックします。

これで設定上はすべて消え去ったはず。

```
$ dig @ns-xxxx.awsdns-xx.com aws.example.com +short
$ dig aws.example.com +short
$
```

おや、もう世界からも消えてしまったようです。もうしばらく残るのかと思ってましたが・・・ホストゾーンを消すと即消えるということか、たまたまそういうタイミングだったのか。

以上です！
