+++
date = "2016-02-24T21:16:42+09:00"
draft = false
title = "CloudFrontでURL振り分けをするときの注意点"
+++

最近仕事で担当しているサービスをオンプレからAWSに移行しました。


ビジネスの都合で短期間で移行しなければいけなかったり、  
AWSに関しては、  
個人的に趣味で少し触った程度だったり、  
会社としてもこれからノウハウを溜めていこうという段階だったので手探り状態で  
大変でしたが周りの方々にフォローしていただきながらなんとか完了することができました。

この記事では移行の際にハマったことを共有します。

<!--more-->

## CloudFront で URL振り分けをする

http://example.com/aaa のアクセスはサーバー(群)A   
http://example.com/bbb のアクセスはサーバー(群)B  
といったかたちで処理を分けたい場合、通常はL7ロードバランサの機能を利用しますが  
ELB にはURLで振り分ける機能がありません。

そのため、代わりに CloudFront に オリジンやビヘイビアを複数設定することで  
L7 スイッチとして使うのが手軽で運用も楽です。

設定についての詳細は参考URLをご参照ください。

#### 参考URL

* <a href="http://mil-o.jp/yb/elb-l7/" target="_blank">[AWS]ELBがURLで振り分けできない問題はCloudFrontでなんとかする</a>
* <a href="https://forums.aws.amazon.com/thread.jspa?threadID=137123" target="_blank">ELBでURLごとにサーバー振り分け</a>

## 注意点

ですがこの方法だと フィーチャーフォン で HTTPS ページが閲覧できなくなります。  
（全ての機種で閲覧できなくなるかはわかりませんが、10機種ほどで確認したら全滅でした）

* フィーチャーフォンはサポートしていない
* HTTPS は利用していない

といった場合は該当しません。

CloudFront はデフォルトではネームベース( SNI )の独自SSL機能を提供していて  
これにはクライアント側も SNI に対応している必要がありますが  
フィーチャーフォンは対応していないようです。  
※ 実際にはフィーチャーフォン以外にも IE6 等の古いブラウザも対応していません。  
　→ <a href="https://ja.wikipedia.org/wiki/Server_Name_Indication#.E5.AF.BE.E5.BF.9C.E7.8A.B6.E6.B3.81" target="_blank">Wikipedia に対応状況がまとまっています</a>

SNI とは SSL/TLS の拡張仕様で、サーバー側が１つのグローバルIPアドレスで複数の証明書を扱えるようになるメリットがあります。

<a href="https://ja.wikipedia.org/wiki/Server_Name_Indication" target="_blank">Server Name Indication - Wikipedia</a>

#### 回避方法

AWSドキュメントにいくつか紹介されています。

<a href="http://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/SecureConnections.html#cnames-https-dedicated-ip-or-sni" target="_blank">CloudFront で HTTPS リクエストを供給する方法の選択</a>  
　→ SNI を使用した HTTPS リクエストの供給（ほとんどのクライアントで動作）

SNI ではなく専用 IPアドレス を利用する方法がありますが、月額で 600 USD(執筆時点) かかるので、フィーチャーフォン対応のために払うにはだいぶ高い金額なのかなと思います。

あとは、ユースケース次第ですが、フィーチャーフォン用のサブドメインを用意して  
フィーチャーフォンでは CloudFront を使わない(URLで振り分けをしない)のも有りだと思います。


#### どうしたか

URLで振り分けるのをやめました。

もともと、オンプレ環境の限られたリソースで  
"aaa にアクセスが集中した場合に bbb に影響を出さないように" という目的で振り分けしていました。  
ですので サーバーA も サーバーB も同じアプリケーションが動いています。

クラウドならスケールアップ／スケールアウトが自由自在なので  
"やばかったら増強すればいい" ということで振り分けるのをやめました。


### 書籍紹介

こちらの本で AWS の基礎を勉強しました。  
各サービスの概要からバックアップ等の実際の運用について書かれていて、  
知りたいところだけをピックアップして読める構成になっているので大変助かりました。


<blockquote class="embedly-card" data-card-key="916e111541fe433792c1330eb7eba55b"><h4><a href="http://www.amazon.co.jp/o/ASIN/4774176737/gihyojp-22">Amazon Web Services実践入門 (WEB+DB PRESS plus)</a></h4><p>Amazon公式サイトでAmazon Web Services実践入門 (WEB+DB PRESS plus)を購入すると、Amazon配送商品なら、配送料無料でお届け。Amazonポイント還元本も多数。Amazon.co.jpをお探しなら豊富な品ぞろえのAmazon.co.jp</p></blockquote>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>

