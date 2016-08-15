+++
date = "2015-09-29T14:24:34+09:00"
draft = false
title = "php で手軽に並列処理をするライブラリ Snidel を作りました"
tags = ["php"]
+++

シルバーウィーク中に php のライブラリを作りました。

<!--more-->

<a class="embedly-card" href="https://github.com/ackintosh/snidel">ackintosh/snidel</a>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>

### Snidel (スナイデル) について

他の言語のマルチスレッド等の並行・並列処理のための機構に近い書き心地で
php で手軽に並列処理をする。というのがコンセプトです。


子プロセス数の制御に <a href="http://php.net/manual/ja/function.msg-get-queue.php" target="_blank">メッセージキュー</a>  
プロセス間のデータのやりとりに <a href="http://php.net/manual/ja/ref.shmop.php" target="_blank">共有メモリ</a>  
を使っています。

命名に特にこだわりは無いのですが、響きがシュッとしてていいかなと思ってます。  
ただ、この記事を書きながらGoogle翻訳にかけてみたらエストニア語で「薬物使用者を注入」って出てきたので少し怖くなってきました...。

![snidel_translate](https://dl.dropboxusercontent.com/u/22083548/octopress/snidel_translate.png)

proc_open() や exec() でコマンドをバックグラウンドで実行するのではなく、  
Callable を別プロセスで実行して、結果を親プロセスが受け取るかたちにしたかったので PCNTL関数 を使うようにしました。

```php
$func = function ($str) {
    sleep(3);
    return $str;
};

$s = time();
$snidel = new Snidel();
$snidel->fork($func, 'foo');
$snidel->fork($func, 'bar');
$snidel->fork($func, 'baz');

var_dump($snidel->get());
// * the order of results is not guaranteed. *
// array(3) {
//   [0]=>
//   string(3) "bar"
//   [1]=>
//   string(3) "foo"
//   [2]=>
//   string(3) "baz"
// }

echo (time() - $s) . 'sec elapsed' . PHP_EOL;
// 3sec elapsed.

```

### 作り始めたきっかけ

仕事で携わっているプロジェクトでは <a href="https://github.com/squizlabs/PHP_CodeSniffer" target="_balnk">PHP_CodeSniffer</a> を使ってコーディング規約チェックをしていて、規約のエラー数を日次で集計して社内の可視化ツールに結果を投げています。

その集計に5分くらいかかっていたので、もっと短時間で終わるようにしたいと思ったのがきっかけです。  
php じゃなくても良かったのですが勉強のため。  
いい感じにできれば他のプロジェクトでも使えるかもですし。

### 結果

集計処理の時間が下記のように改善しました。

- ビフォー

```
real 5m9.580s
user 3m3.123s
sys  0m14.849s
```

- アフター

```
real 2m44.248s
user 3m1.867s
sys  0m20.341s
```

実行時間が45%くらい削減できました。  
また、実行時間よりユーザーCPU時間の方が長くなっているので
複数のCPUが使えてることがわかります。

ライブラリの出来としては、github の issues に挙げていますが  
まだ課題があったりするのでもう少しいじりながら楽しみたいと思います。
