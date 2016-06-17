+++
date = "2015-11-08T16:05:24+09:00"
draft = false
title = "Snidel 0.2 をリリースしました"
tags = ["php"]
+++

<a href="https://github.com/ackintosh/snidel" target="_blank">Snidel</a> バージョン 0.2 をリリースしました。  
この記事は、  
追加した３つの機能の紹介と、Snidel を使ってもらって嬉しかった！の話になります。

個人的に、(実際のアプリケーションで必要とされるかは別として）面白い試みをした機能もありますので興味を持っていただけると嬉しいです。

<!--more-->

### 特定の処理結果を取得

処理結果を取得するメソッドとして `Snidel::get()` を用意していますが  
並列に処理する関係で、 `Shidel::get()` で得られる結果の順番は保証されません。

```php
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

```

ですので、特定（１つまたは複数）の処理結果を取得したいケースに対応できるように  
処理をフォークする時／結果を取得する時にタグを指定できるようにしました。

```php
$snidel->fork($func, 'foo', 'tag1');
$snidel->fork($func, 'bar', 'tag1');
$snidel->fork($func, 'baz', 'tag2');

var_dump($snidel->get('tag1'));
// array(2) {
//   [0]=>
//   string(3) "foo"
//   [1]=>
//   string(3) "bar"
// }
```

上記サンプルコードでは `tag1` を指定した処理の結果だけを取得しています。

### ログの出力

これは Snidel 自体の開発で必要性を感じて実装しました。

なかなか意図した動作をしない時に、どのプロセスが・どの順番で・いつ生成されたのかを  
追跡する手段がないと辛いですよね。

最低限の機能しかない頃は echo しまくるだけで間に合ってたのですが  
機能追加を重ねるごとに辛くなってきたので、ログの出力先として resource をセットできるようにしました。

```php
$fp = fopen('php://stdout', 'w');
$snidel->setLoggingDestination($fp);

$snidel->fork($func, 'foo');
// [info][26304(p)] created child process. pid: 26306
// [info][26306(c)] waiting for the token to come around.
// ...

```

上記サンプルコードでは標準出力にプロセス生成などのログが出力されています。

### 複数の処理を並列につなげて実行

個人的に、(実際のアプリケーションで必要とされるかは別として）面白い試みをした機能です。 

A、B、C ３つの処理を続けて処理する（Aの結果をBが処理し、Bの結果をCが処理する）ケースで  
３つの処理それぞれを並列に実行することができます。  
↑を読んでも、いまいちイメージが伝わらないと思いますがとりあえず使い方を書いていきます。

まず `Snidel::map()` に最初の処理Aと、処理Aを適用する配列を渡します。

```php
$map = $snidel->map($args, function ($arg) {
    // do something. (A)
});
```

そして `then()` で後続の処理を指定していきます。

```php
$map->then(function ($arg) {
    // do something. (B)
})->then(function ($arg) {
    // do something. (C)
});
```

最後に `Snidel::run()` で、定義した処理を実行します。

```php
$snidel->run($map);
```

例として、スペース区切りの文字列をキャメルケースに変換する処理を記載します。

```php
$args = [
    'BRING ME THE HORIZON',
    'ARCH ENEMY',
    'BULLET FOR MY VALENTINE',
    'RACER X',
    'OF MICE AND MEN',
    'AT THE GATES',
];

$snidel = new Snidel($maxProcs = 2);

// each of the functions are performed in parallel.
$camelize = $snidel->map($args, function ($arg) {
    return explode(' ', strtolower($arg));
})->then(function ($arg) {
    return array_map('ucfirst', $arg);
})->then(function ($arg) {
    return implode('', $arg);
});

var_dump($snidel->run($camelize));
// array(6) {
//   [0] =>
//   string(6) "RacerX"
//   [1] =>
//   string(20) "BulletForMyValentine"
//   [2] =>
//   string(9) "ArchEnemy"
//   [3] =>
//   string(17) "BringMeTheHorizon"
//   [4] =>
//   string(10) "AtTheGates"
//   [5] =>
//   string(12) "OfMiceAndMen"
// }
```

こんな感じの実行イメージです。  
青 : 処理A (explode)  
緑 : 処理B (ucfirst)  
黄 : 処理C (implode)  
<iframe src="https://player.vimeo.com/video/144969743" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

処理Aが並列数2で開始して 結果を処理Bに渡した後、  
処理Bの開始と同時に、次の処理Aも開始します。
（文章が下手ですいません...）

なお、 **下記ではありません** 。

<iframe src="https://player.vimeo.com/video/145030564" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

各処理が1秒かかる場合、  
前者が 5秒 で終了するのに対して、後者は 9秒 かかってしまいます。

ただ、処理は早く終わりますが実際の並列実行数は Snidel のコンストラクタで指定する数より多くなります。  
（例でいえば最大 6になります）

以上が追加した機能になります。

### 社内で使ってもらえました!

初回リリース時の記事 (<a href="/blog/2015/09/29/snidel/">php で手軽に並列処理をするライブラリ Snidel を作りました</a>) で  

> いい感じにできれば他のプロジェクトでも使えるかもですし。

と書いていたのですが、社内slackで作ってみました〜と静かにアピールしたら  
実際に使ってもらえて、けっこう成果が出たみたいでめっちゃ嬉しかったです。  
この記事で紹介した機能はその勢いで実装しました。

今後は...  
初回リリースから１ヶ月ちょっと経って、実装したい機能は一通り終わったのですが、  
テストコードが少なくてちょっと恥ずかしい感じなので  
引き続きそのへんをいじっていきたいと思っています。
