+++
date = "2013-06-11T16:41:31+09:00"
draft = false
title = "PHP用のベンチマークツールを作りました"
tags = ["php"]
+++

こちらの記事に影響を受けて、参考にさせていただきながら自分でも作ってみました。

<a href="http://blog.yuyat.jp/archives/1063" target="_blank">PHP 用ベンチマーキングフレームワーク Joshimane というのを作った</a>

<!--more-->

自分はなかなかいい名前が思いつかなかったので Benchy にしました。

<a href="https://github.com/ackintosh/benchy" target="_blank">https://github.com/ackintosh/benchy</a>

PEAR::Benchmarkと比べるとモダンな感じかなぁと思っています。

シンプルさと拡張性の高さをウリにできるように考えました。

#### インストール
もちろんComposerでインストールできます。

```composer.json
{
  "require": {
    "ackintosh/benchy": "dev-master"
  }
}
```
```
$ php composer.phar install
```

#### 使い方
```php
<?php
require_once 'vendor/autoload.php';
$reporter = Ackintosh\Benchy::run(function ($reporter) {
    // do something
    echo $reporter->time->elapsed() . PHP_EOL;
    // do something
    echo $reporter->time->elapsed() . PHP_EOL;
}, 1000); // runs 1,000 times.(default : 1 )
echo 'total : ' . $reporter->time->total() . PHP_EOL;
echo 'average : ' . $reporter->time->average() . PHP_EOL;
```
これで途中経過の時間と、合計・平均時間がわかります。

### 拡張性
Ackintosh/Bechy/Marker ディレクトリにクラスを配置してください。

```
<?php
class Example extends AbstractMarker
{
        public function hoge() { 'fuga'; }
}
```
そうするとReporterクラスで使えるようになります。

```
<?php
$reporter = Ackintosh\Benchy::run(function ($reporter) {
    // do something
    echo $reporter->example->hoge();// fuga
});
echo $reporter->example->hoge();// fuga
```

### フックポイント
- before：　ベンチマーク開始前
- after：　ベンチマーク終了後
- before_per_laps：　ベンチマーク前（毎回）
- after_per_laps：　ベンチマーク後（毎回）
