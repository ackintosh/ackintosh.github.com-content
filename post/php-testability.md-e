+++
date = "2013-10-18T17:14:32+09:00"
draft = false
title = "グローバル関数への依存を排除してテスタビリティを上げる"
tags = ["php", "test"]
+++

テストしにくい状況って色々な原因があると思いますが、  
今回はグローバル関数への依存について。

<!--more-->

例えば下記のコードでは  
receiveDataメソッドの中でmail関数を呼び出しているので  
テストしにくくなっています。  
（テストは書けるけどテスト走らせる度にメールが飛ぶのはアレですね）  

<script src="https://gist.github.com/ackintosh/7026140.js?file=1.php"></script>

<a
href="http://www.amazon.co.jp/%E3%83%AC%E3%82%AC%E3%82%B7%E3%83%BC%E3%82%B3%E3%83%BC%E3%83%89%E6%94%B9%E5%96%84%E3%82%AC%E3%82%A4%E3%83%89-Object-Oriented-SELECTION-%E3%83%9E%E3%82%A4%E3%82%B1%E3%83%AB%E3%83%BBC%E3%83%BB%E3%83%95%E3%82%A7%E3%82%B6%E3%83%BC%E3%82%BA/dp/4798116831"
target="_blank">レガシーコード改善ガイド</a>では  
グローバル関数の部分をインスタンスメソッドにして、処理をグローバル関数にまるっと委譲することで、接合部を作る方法が紹介されています。  
例えばこんな感じでしょうか。  

<script src="https://gist.github.com/ackintosh/7026140.js?file=2.php"></script>

接合部となったメソッドをサブクラスでオーバーライドしてテストしてます。  

ただ、わざわざサブクラスを定義するのも面倒な気もするし  
もう少しカジュアルな方法がないかなということで  

<script src="https://gist.github.com/ackintosh/7026140.js?file=3.php"></script>

テストのためにややプロダクションコードが増えますがメソッドの差し替えができるようになりました。

ちなみに、無名関数の中でアサーションが書けるのでmail関数が受け取る引数をアサートすることもできます。

こんな感じです。

<script src="https://gist.github.com/ackintosh/7026140.js?file=4.php"></script>

ここまで書いておいてあれですが、こちらにもっと良い方法が解説されていました。  
<a href="http://phpmentors.jp/post/46982737824" target="_blank">PHPメンターズ -> 時計オブジェクト（ドメインクロック）を導入してテスト容易性と意図性を高める</a>
