+++
date = "2013-04-18T16:31:56+09:00"
draft = false
title = "PHPでTCPサーバー"
tags = ["php"]
+++

PHPでTCPサーバーを書いてみました。

pcntl関数を使うには、phpソースをbuildする時に–enable-pcntlを付けないといけません。

<!--more-->

個人的には、pcntl_fork()したあとswitch文で分岐する流れが、  
理解するのに苦労しました。

ちょうどRubyで並列処理を勉強していたのですが、  
やっぱりRubyの方が直感的で書きやすいですね。  

<a
href="http://melborne.github.io/2011/09/29/irb-Ruby-fork-WebSocket/"
target="_blank">irbから学ぶRubyの並列処理 ~ forkからWebSocketまで</a>

<script src="https://gist.github.com/ackintosh/5381925.js"></script>
