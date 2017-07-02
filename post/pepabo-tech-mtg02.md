+++
date = "2017-06-30T09:22:31+09:00"
draft = false
title = "第2回 EC 事業部 Tech MTG"

+++

ブログに書くのが遅くなってしまった。

<!--more-->


[前回](https://ackintosh.github.io/blog/2017/03/09/pepabo-tech-mtg01/)につづいて今回も発表した。  
前回のつづきのような感じになっていて、API を中心にしたアーキテクチャに移行していくにあたって、まずは API クライアントのカオスな状況を整備(してから CircuitBreaker を入れたり)したいという内容。


<script async class="speakerdeck-embed" data-id="31f5262e7a124f11a378df3a96bdb7a7" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>


SwaggerCodegen 生成したクライアント(API クラス、モデルクラス)を使うけども、生成したままのコードではサービスに投入するにはまだ不十分だと思うところがあって、その辺りをどうしていこうかというのを後半に書いている。

( 生成した API クラスで CircuitBreaker を使う方法を以前 [Swagger Codegen + CircuitBreaker(Ganesha)](https://ackintosh.github.io/blog/2017/04/09/swagger-codegen-with-ganesha/) に書いていた )

<!--
モデルクラスにメソッドを追加したくなるケースがあると思っていて、スライドではそれが伝わらないので補足したい。  
(発表では、実際に現状のソースコードで具体的な事例を示したが、もちろん公開するスライドには載せられないので、下記のざっくりしたサンプルコードで雰囲気を感じ取っていただければ)
-->



Generation Gap パターンの適用方法については、スライドに書いてる方法だとちょっと微妙なことに気づいたので考え中。


ひとまず発表が終わってホッとしたけど、まだ願望を共有しただけなのでこれからが本番。



(会社のブログ : [5月病を吹きとばせ！第2回EC事業部TechMTGを開催しました！ - ペパボテックブログ](http://tech.pepabo.com/2017/05/29/ec-tech-mtg-2nd-report/))