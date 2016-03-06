+++
date = "2013-04-29T15:51:23+09:00"
draft = false
title = "FuelPHPに独自のバリデーションルールを追加する"
tags = ["php"]
+++

<a href="http://fuelphp.com/" target="_blank">FuelPHP » A simple,
flexible, community driven PHP5.3 framework.</a>  
<a href="http://fuelphp.jp/">FuelPHP.JP 日本語ドキュメント</a>

実際の開発では、独自のバリデーションルールがいくつか必要になります。  
FuelPHPで追加する方法のメモです。φ(｀д´)ﾒﾓﾒﾓ…

<!--more-->

#### バリデーションルールを定義する
<script src="https://gist.github.com/ackintosh/5479927.js?file=gistfile1.php"></script>


#### 定義したルールのテストを書く
<script src="https://gist.github.com/ackintosh/5479927.js?file=gistfile2.php"></script>

#### ルールを適用する
<script src="https://gist.github.com/ackintosh/5479927.js?file=gistfile3.php"></script>

#### エラーメッセージを定義する
<script src="https://gist.github.com/ackintosh/5479927.js?file=gistfile4.php"></script>
