+++
date = "2017-09-07T09:16:11+09:00"
draft = false
title = "udemy で Learn Apache Kafka for Beginners を修了した"
+++

[Apache Kafka Series - Learn Apache Kafka for Beginners | Udemy](https://www.udemy.com/apache-kafka-series-kafka-from-beginner-to-intermediate/)

<!--more-->

![](https://udemy-certificate.s3.amazonaws.com/image/UC-45ZK5CZX.jpg)

### きっかけ

[Snidel](https://github.com/ackintosh/snidel) という PHP のライブラリを開発していて、メッセージングやプロセスが協調して何かをやることに興味が湧いていた。そんな中、[Apache Kafkaに入門した | SOTA](http://deeeet.com/writing/2015/09/01/apache-kafka/) や [サイボウズのサービスを支えるログ基盤](https://www.slideshare.net/ShinyaUeoka/ss-78228751) といったコンテンツに出会い、Kakfa 面白そう！ということでインターネットを徘徊していたら udemy のコースを見つけた。

### 感想

コースの概要やコンテンツは上記リンクを見ていただければわかるので、いくつかピックアップして感想を。

##### Core Concepts


Kafka のコンセプトや用語・仕組みから、一般的なメッセージ配信に関するセマンティンクスまで、大事なことをギュッと一気に解説される。このセクションの資料は繰り返し見返すことになりそう。

##### Hands-on Practice

単純に「○○したいときは××コマンドを実行する」という解説ではなく、最初にコマンドにあえて何もオプションを指定せず、必須パラメータエラーを出しながらインクリメンタルに説明を進めていくのがとてもわかりやすかった。

##### Topic Configuration


実践的な内容。
実際に Kakfa を使って構築しようとしたときにぶつかるであろう、"パーティションの数どうする？" "レプリケーションファクターの数は？" といった疑問や、パフォーマンスを考慮するとデフォルトから変更した方がいい設定値について明確なガイドラインを示してもらえた。

LogCompaction について、文章だけでは誤解や疑問が残ってしまうがハンズオン形式で実際に動かしながら挙動を見れたのでしっかり理解できた。

---

今回受講したのは [Stephane Maarek](https://www.udemy.com/apache-kafka-series-kafka-from-beginner-to-intermediate/#instructor-1) さんの Apache Kafka Series の一番基礎のコース。引き続きシリーズのコースを受講していきます。