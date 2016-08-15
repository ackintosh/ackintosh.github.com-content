+++
date = "2016-08-05T15:34:21+09:00"
draft = false
title = "ngx_mruby でシャドウプロキシ"
+++

最近 ngx_mruby をいじり始めて「ビルドできた！」「おぉ、簡単にWebサーバーの振る舞いを変えられる！」ところまでいったので、  
何を作ろうかと考えて初めに思いついたのがシャドウプロキシでした。

<!--more-->

とはいえシャドウプロキシが何なのかはよくわかっていなかったので予め下記を参考にさせていただきました。

**シャドウプロキシサーバーの実装**

* <a href="https://github.com/cookpad/kage" target="_blank">kage</a>
* <a href="https://github.com/kentaro/delta" target="_blank">delta</a>
* <a href="https://github.com/lestrrat/p5-Geest" target="_blank">Geest</a>  
 
**シャドウプロキシの仕組み**

* <a href="http://qiita.com/toyama0919/items/ae4bc88423317e6668b1" target="_blank">fluentdで本番環境を再現する</a>
* <a href="http://qiita.com/kamatama_41/items/dc12d5127e7de9a1849b" target="_blank">Shadow Proxyのご紹介 at Quipper</a>

まとめると、

* 本番のリクエストをバックエンド（Shadow環境）に複製することで、本番に限りなく近い状態を再現できる。
* それによって、いきなり本番投入するのが怖い何かを安全に試運転させてバグの検知ができる。
* 心穏やかにリリースに臨める。

#### mruby-ngx-shadow_proxy


ざっくりした実装しかしていませんが、github で公開しています。

<blockquote class="embedly-card" data-card-key="916e111541fe433792c1330eb7eba55b" data-card-type="article"><h4><a href="https://github.com/ackintosh/mruby-ngx-shadow_proxy">ackintosh/mruby-ngx-shadow_proxy</a></h4><p>mruby-ngx-shadow_proxy - HTTP shadow proxy using ngx_mruby.</p></blockquote>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>


mrbgem を作るのは初めてだったのですが、<a href="https://github.com/matsumoto-r/mruby-mrbgem-template" target="_blank">mruby-mrbgem-template</a> を使うと必要なディレクトリ構造やファイルを生成してくれるので便利でした。

<blockquote class="embedly-card" data-card-key="916e111541fe433792c1330eb7eba55b" data-card-type="article-full"><h4><a href="http://blog.matsumoto-r.jp/?p=3923">mrubyの拡張モジュールであるmrbgemのテンプレートを自動で生成するmrbgem作った</a></h4></blockquote>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>


##### 試行錯誤

実装を始めた当初は、上記に挙げたサーバーのように ngx_mruby で本番とバックエンドそれぞれに並列にリクエストを複製しようとしたのですが下記理由で別の方法にしました。

* 漠然とした違和感（何か ngx_mruby らしい方法があるのではないか）
* できるだけフロントに手を入れたくない
* そもそも mruby がスレッドセーフではない
  * この辺がまだ理解できていないので語弊があるかもしれません
  * 参考 https://github.com/mruby/mruby/issues/1657
  * 実際に <a href="https://github.com/mattn/mruby-thread" target="_blank">mattn/mruby-thread</a> を使って試行錯誤していたのですがスレッド間のデータのやりとりがうまくできませんでした

なので最終的に、バックエンドにリクエストを複製するだけにしました。

<a href="https://github.com/ackintosh/mruby-ngx-shadow_proxy#example" target="_blank">README<a/>に例を載せていますが、  
ngx_mruby には<a href="https://github.com/matsumoto-r/ngx_mruby/wiki/Directives#ngx_mruby-http-module-directives" target="_blank">いろんなディレクティブ</a>があって nginx のフェーズにフックして制御できるようになっているので、
mruby_log_handler というディレクティブを使うことで、クライアントにレスポンスを返した後のフェーズでバックエンドにリクエストを複製するようにしています。


