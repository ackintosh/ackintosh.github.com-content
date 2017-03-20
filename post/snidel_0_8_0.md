+++
date = "2017-03-10T21:48:28+09:00"
draft = false
title = "Snidel 0.8 をリリースしました"
tags = ["php"]
+++


Snidel 0.8 をリリースしました。2点、変更内容やその理由をご紹介します。

<!--more-->

<blockquote class="embedly-card" data-card-key="916e111541fe433792c1330eb7eba55b" data-card-image="https://github.com/ackintosh/snidel/raw/master/images/snidel-queue-sqs.png" data-card-type="article"><h4><a href="https://github.com/ackintosh/snidel">ackintosh/snidel</a></h4><p>snidel - A multi-process container.</p></blockquote>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>

## 独自のキューを実装・利用できるようになりました

今回のバージョンアップで、内部で使っているキューを自由に差し替えられるようになりました。  
下記のように Amazon SQS を利用することもできます。

![with Amazon SQS](https://github.com/ackintosh/snidel/raw/master/images/snidel-queue-sqs.png)

##### なぜ？

Snidel は、ワーカープロセスとのデータ(ワーカーに実行してほしい処理や、実行した結果)の受け渡しにキューを使っていて、これまでは System V キューに決め打ちになっていました。  
そのため、Snidel を使って大きな処理をたくさん走らせたい場合に、(OSの設定によりますが)System V キューは扱えるデータサイズが限られているので、そこがネックになってしまうことがありました。

今回の変更で、よりスケーラブルなキューを利用して大規模な並列処理を実行できるようになりました。

##### 独自のキューを実装・利用するには？

所定のインターフェースを実装して、コンストラクタに渡すパラメータでクラス名を指定してください。

例: [ackintosh/snidel-queue-sqs: queue plugin for snidel.](https://github.com/ackintosh/snidel-queue-sqs)


キューの初期化等に必要なパラメータ(Amazon SQS の例でいうところのクレデンシャル情報)は、Snidel のコンストラクタに任意のキーで指定しておけば、キューのコンストラクタに渡される Config オブジェクトから利用できます。

## アーキテクチャを変更しました

キューをプラガブルにするのに伴ってアーキテクチャを変更しました。  

###### ビフォー

![before](https://github.com/ackintosh/snidel/raw/master/images/0.6_master_worker.png)

###### アフター

![after](https://github.com/ackintosh/snidel/raw/master/images/0.8_pluggable_queue.png)

これまでは、マスタープロセスがキューを通じてタスクを受け取って、ワーカーを fork していましたが  
今回の変更で、マスタープロセスは予めワーカーを prefork しておき、ワーカーがキューからタスクを受け取るかたちになりました。

##### なぜ？

プロセスを fork するというのは分身を作ることなので、例えば外部ストレージに接続中のプロセスが fork した場合、ストレージ接続も子プロセスにコピーされます。そのため、親(または子)プロセスがストレージ接続を切った場合に、もう片方のプロセスの接続が影響を受けてしまう(動作が不安定になる)可能性があります。  
Snidel の場合キューの接続で上記の可能性があり、これまで System V キューでは問題ありませんでしたが、様々なストレージをキューとして利用できるようになることで問題が表面化してしまいます。それを避けるためにこの変更を行いました。

Snidel は3種類のプロセスが協調して並列処理を行います。今回の変更によりそれぞれの役割が下記のようになり、マスタープロセスがキューに接続する必要が無くなったので安定して動作するようになりました。

- メイン
  - マスターを fork する
  - ワーカーに処理させたいタスクをキューに入れる
  - ワーカーの処理結果をキューから受け取る
- マスター
  - ワーカーを fork する
  - ワーカーの起動数を制御する
- ワーカー
  - キューからタスクを受り、実行し、結果をキューに入れる

## 今後

PHP5.6 未満のサポート終了を予定しています。  
これにより、5.3 用に冗長になっていたコードをスリムにできたりジェネレーターを使って省メモリにできるかなと考えています。

また、いつでもプルリクエストを歓迎していますので、気になったところや良いアイデアをお持ちのかたは是非よろしくお願いします。