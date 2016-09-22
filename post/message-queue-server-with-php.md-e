+++
date = "2013-01-03T16:47:11+09:00"
draft = false
title = "ZendQueueとKestrelでメッセージキューサーバーを体験"
tags = ["php"]
+++

#### Kestrel
Scalaで書かれたメッセージキューサーバー。Twitterで使われてるらしいです。  
<a href="http://samuraism.jp/diary/2011/11/20/1321770660000.html" target="_blank">Twitterで使っているScalaで書かれたオープンソースのメッセージキューサーバー、Kestrel
:侍ズム#samuraism</a>

<!--more-->

#### インストールと起動
```
$ curl -O http://robey.github.com/kestrel/download/kestrel-2.4.1.zip
$ unzip kestrel-2.4.1.zip
$ cd kestrel-2.4.1
$ sudo java -jar kestrel_2.9.2-2.4.1.jar
```

#### ZendQueue
Zend Frameworkのコンポーネントの１つで、メッセージキューを利用するために使います。　　

<a href="https://github.com/zendframework/ZendQueue" target="_blank">https://github.com/zendframework/ZendQueue</a>

メッセージを格納する方法によって複数のアダプタが用意されています。  
Kestrel用のアダプタはありませんが、Kestrelはmemcachedプロトコルをサポートしているので、MemcacheQアダプタを利用します。  

#### Memcache
あらかじめMemcachedライブラリもインストールしておいて下さい。  
Macの場合はHomebrewを使うと簡単にインストールできます♪  

```
$ brew install memcached
$ brew install memcache-php
```
#### メッセージキューサーバーを体験
２つのスクリプトを用意してください。  
・worker.php :
ワーカープロセス。キューからメッセージを取得して表示する。
<script src="https://gist.github.com/ackintosh/4444078.js?file=worker.php"></script>
・front.php : キューにメッセージを送信する。
<script src="https://gist.github.com/ackintosh/4444078.js?file=front.php"></script>

ターミナルを２つたちあげてください。  
・ターミナル１でworker.phpを実行  
プロンプトが返ってこない → Kestrelのキューを監視してくれています。  

・ターミナル２でfront.phpを実行すると…  
ターミナル１に「Hello, World!」と表示されます！  

簡単ですが以上です。  
Hello, Worldが表示された時には感動しますね (・∀・)  
worker.phpを実行するターミナルを増やしたりするとなお楽しくなってきます♪
