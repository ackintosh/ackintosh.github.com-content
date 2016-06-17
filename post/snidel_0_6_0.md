+++
date = "2016-05-04T02:16:17+09:00"
draft = false
title = "Snidel 0.6 をリリースしました"
+++

外部的なインターフェースは変わっていませんが、  
内部的なアーキテクチャを大幅に変更しました。

<a href="https://github.com/ackintosh/snidel/releases/tag/0.6.0">Release 0.6.0 · ackintosh/snidel</a>

<!--more-->

### 従来のアーキテクチャ

![original_architecture.png](https://raw.githubusercontent.com/ackintosh/snidel/master/images/original_architecture.png)

* Snidel::fork() を実行したタイミングで子プロセスを fork し、
生成された子プロセスはトークンが回ってくるまで待機する  
（同時実行数を制御するために Snidel\Token オブジェクトを使用）
* 予め設定された同時実行数以下までの子プロセスは即座に処理が開始されるが、それ以降の子プロセスは先に実行中のプロセスが完了するまでは待機状態になる
* 子プロセスは処理を実行して結果を共有メモリに保存し、最後に(Snidel::__destruct()で)トークンを返却する
* Snidel::wait() または Snidel::get() を実行することで親プロセスが、各子プロセスと対になる共有メモリから実行結果を取得する  
(実行中・待機中の子プロセスがあれば、それらが完了するまで親プロセスは待機する)

### 問題点

Snidel::fork() で即座に子プロセスを fork するため、  
場合によっては瞬間的に大量の fork が発生するため下記の問題がありました。

* fork の負荷
* 大量の待機プロセスが発生するためプロセス数の上限に達してしまう可能性

上記を避けるために単純に fork 数をコントロールしようとすると  
同時実行数を超えた Snidel::fork() 呼び出しの際に親プロセス (Snidel を利用する側) を  
実行中の子プロセスが終了するまで待機させなければいけません。

### 改善後のアーキテクチャ

Master-Workerモデルに変更しました。

![master_worker.png](https://raw.githubusercontent.com/ackintosh/snidel/master/images/master_worker.png)

* 初回の Snidel::fork() を実行したタイミングで Master プロセスを fork する
* 実行したい処理は専用のキュー(Task Queue)に追加する
* Master プロセスが Task Queue から処理を取り出し、それを実行する Worker プロセスを fork する  
（Worker の同時起動数を制御するために Snidel\Token オブジェクトを使用）
* Worker プロセスが実行結果を専用のキュー(Result Queue)に追加し、トークンを返却する
* Snidel::wait() または Snidel::get() を実行することで、Result Queue から実行結果を取得する  
(実行中・未実行のタスクがあれば、それらが完了するまで待機する)

これにより、親プロセス(内部的には Owner プロセスと呼んでいます)は  
実行したい処理をキューに追加するだけで良くなって、  
あとは Master プロセスが Worker の同時起動数に気を配りながらイイ感じに進めてくれます。  
もちろん、Master-Worker が働いている間 Owner は他の仕事ができます。

改善内容については以上になります。

ただ現状、従来のアーキテクチャを完全に置き換えたわけではなく、  
Snidel::map() に関連する処理の中で内部的に利用されていて、今後こちらも置き換える予定です。  
Snidel::map() はバージョン0.2で実装した機能で、リリース時の記事で「複数の処理を並列につなげて実行」という項目で紹介しています。  
[Snidel 0.2 をリリースしました](http://ackintosh.github.io/blog/2015/11/08/snidel_0_2_0/)

