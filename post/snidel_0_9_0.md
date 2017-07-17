+++
date = "2017-07-17T19:11:54+09:00"
draft = false
title = "Snidel 0.9 をリリースしました"
tags = ["php"]
+++

[前回のリリース](/blog/2017/03/10/snidel_0_8_0/)はアーキテクチャ等の内部的な変更がメインでしたが、今回は逆にライブラリのインターフェースを大きく変える変更を加えています。

<!--more-->

## PHP 5.6 未満のサポートを終了

永らく古いバージョンに対応してきましたが、この度 5.6 未満のサポートを終了しました。  
これにより 5.3 用に冗長になっていたコードがスマートになったり、後述する機能改善を進めることができました。

## PSR-3 Logger Interface をサポート

PHP-FIG が定めるロガーのインターフェース規約である [PSR-3 Logger Interface](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md) をサポートしました。

#### 従来

これまでは、 Snidel のインスタンスにログの出力先として resouce を渡すだけの簡素な実装でした。

```php
$snidel->setLogDestination(fopen('php://stdout', 'w'));
$snidel->fork($func, 'foo');

// [2015-12-01 00:00:00][info][26304(p)] created child process. pid: 26306
// [2015-12-01 00:00:00][info][26306(c)] --> waiting for the token to come around.
// [2015-12-01 00:00:00][info][26306(c)] ----> started the function.
// [2015-12-01 00:00:00][info][26306(c)] <-- return token.
// ...
```

#### ver0.9

PSR-3 に準拠したロガーを利用できます。お好みのロガーを利用いただけるようになり、従来と比べて出力先等の自由度が上がりました。

```php
use Monolog\Formatter\LineFormatter;
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

$monolog = new Logger('sample');
$stream = new StreamHandler('php://stdout', Logger::DEBUG);
$stream->setFormatter(new LineFormatter("%datetime% > %level_name% > %message% %context%\n"));
$monolog->pushHandler($stream);

$snidel = new Snidel([
    'logger' => $monolog,
]);
$snidel->process($f);

// 2017-03-22 13:13:43 > DEBUG > forked worker. pid: 60018 {"role":"master","pid":60017}
// 2017-03-22 13:13:43 > DEBUG > forked worker. pid: 60019 {"role":"master","pid":60017}
// 2017-03-22 13:13:43 > DEBUG > has forked. pid: 60018 {"role":"worker","pid":60018}
// 2017-03-22 13:13:43 > DEBUG > has forked. pid: 60019 {"role":"worker","pid":60019}
// 2017-03-22 13:13:44 > DEBUG > ----> started the function. {"role":"worker","pid":60018}
// 2017-03-22 13:13:44 > DEBUG > ----> started the function. {"role":"worker","pid":60019}
// ...
```

## 処理結果の取得にジェネレータを利用しメモリ使用量を低減

#### 従来

各ワーカープロセスが処理したすべての結果を配列(コレクションオブジェクトのプロパティ)に溜め込んでいたため、処理数に比例してメモリ使用量が大きくなってしまう課題がありました。

```php
// Snidel::get() returns instance of Snidel\Result\Collection
$collection = $snidel->get();

// Snidel\Result\Collection implements \Iterator
foreach ($collection as $result) {
    echo $result->getFork()->getPid();
    echo $result->getOutput();
    echo $result->getReturn();
}
```

#### ver0.9

PHP5.5 で実装された [ジェネレータ](http://php.net/manual/ja/language.generators.overview.php) を利用することでメモリ使用量を低減しています。  
また、当改修に伴いメソッド名を `Snidel::get()` から `Snidel::results()` に変更しています。

```php
// `Snidel::results()` returns `\Generator`
foreach ($snidel->results() as $r) {
    echo $r->getProcess()->getPid();
    echo $r->getOutput();
    echo $r->getReturn();
}
```

シンプルな 10,000 個の処理を、並列数 20 で実行した場合に下記のようにメモリ使用量の違いがでました。

| バージョン | メモリ使用量 |
|:-----------:|------------:|
| 0.8 | 29.29 MB |
| 0.9 | 9.66 MB |


## その他の変更点

ワーカープロセスに処理を渡すための `Snidel::fork()` を `Snidel::process()` にリネームしました。

```php
$snidel->fork(function ($arg) { echo $arg; }, 'foo');
```

```php
$snidel->process(function ($arg) { echo $arg; }, 'foo');
```

初期の実装では `Snidel::fork()` は子プロセスをフォークしていたのですが  
[ver0.6](https://ackintosh.github.io/blog/2016/05/04/snidel_0_6_0/)で Master-Worker モデルにアーキテクチャを変更したタイミングで、プロセスをフォークする代わりに、タスクをキューに追加するだけの実装になっており、メソッド名から連想する内容と相違がある状態でした。

そんな中で今回、「ジェネレータを利用してメモリ消費を低減」でご紹介したとおり、処理結果を取得するメソッドの名前を変更したので、ちょうどいい機会かなと思い、それと対になる `Snidel::fork()` の方も変更しました。


## 今後

ワーカープロセスがタスクを処理する際のエラーハンドリングが雑なままなので改善する予定です。  
また、ワーカープロセスとのデータ受け渡しで利用しているキューの部分を外部ライブラリを使うように変更して、 Snidel 自体は並列処理の部分に集中できるようにしていければと考えています。