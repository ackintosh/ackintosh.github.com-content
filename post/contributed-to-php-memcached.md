+++
date = "2017-04-29T12:23:10+09:00"
draft = false
title = "php-memcached にコントリビュートしたので経緯とかを書き留めておく"
tags = ["php"]
+++

Fix optional parameter getStats($type) by ackintosh · Pull Request #337  
https://github.com/php-memcached-dev/php-memcached/pull/337

たった4行のちょっとした修正だけど経緯とかを書き留めておく。

<!--more-->

## やったこと

### Memcached::getStats() にドキュメントに書いてない引数があった

2017-04-29 時点では `Memcached::getStats()` の [ドキュメント](http://php.net/manual/ja/memcached.getstats.php) には引数の記載がないが、実際は `type` という引数が [#298](https://github.com/php-memcached-dev/php-memcached/pull/298) で追加され、 v3.0.0 でリリースされている。省略可能なので全く気づかなかった。

#### 引数 `type` について

この引数は、lib_memcached の [memcached_stat_execute](http://docs.libmemcached.org/memcached_stats.html#memcached_stat_execute) 関数の第2引数に渡される。

libmemcached 1.1.0 documentation > memcached_stat_execute

> args is an optional argument that can be passed in to modify the behavior of “stats”. You will need to supply a callback function that will be supplied each pair of values returned by the memcached server.


[memcached のドキュメント](https://github.com/memcached/memcached/blob/master/doc/protocol.txt#L583-L588) には stats の引数について下記のように書かれている。

memcached > Protocol > Statistics

> Depending on \<args>, various internal data is sent by the server. The
kinds of arguments and the data sent are not documented in this version
of the protocol, and are subject to change for the convenience of
memcache developers.

この引数について把握することが目的ではなかったので、ちょっと試してみただけで、挙動ついては正直理解していない。

```php
var_dump($memcached->getStats('sizes_disable'));

// array(1) {
//   'localhost:11211' =>
//   array(1) {
//     'sizes_status' =>
//     string(8) "disabled"
//   }
// }
```

### ReflectionParameter でオプショナルだと認識されてなかったので修正した

ドキュメントに書かれてないだけなら フ～ン で終わるが、省略可能なのに `ReflectionParameter::isOptional()` が false を返していたので、直さねばという気持ちになった。

```php
$method = new \ReflectionMethod(new \Memcached(), 'getStats');
foreach ($method->getParameters() as $p) {
    var_dump($p->isOptional());
}
// bool(false)
```

いろいろと調べながらやっていたので、かなり時間がかかってしまったのだが  
結局やったことは [`ZEND_BEGIN_ARG_INFO` から `ZEND_BEGIN_ARG_INFO_EX` に置き換えただけ](https://github.com/php-memcached-dev/php-memcached/pull/337/files)。

実行時は `zend_parse_parameters` の第2引数に渡してる型指定子 [`"|S!"`](https://github.com/php-memcached-dev/php-memcached/blob/6b660bd554368bae69c2fbf490c6e78712e50708/php_memcached.c#L2645) に従ってパースされるが、リフレクションでは上記マクロで定義した情報が参照されるようだ。（このへんはまだ理解が足りていない）

[`ZEND_BEGIN_ARG_INFO_EX` マクロ](https://github.com/php/php-src/blob/PHP-7.1.0/Zend/zend_API.h#L115) は第4引数で必須の引数の数を指定できるので、これを 0 にすると `ReflectionParameter::isOptional()` が true を返すようになった。


## 経緯

`ReflectionParameter::isOptional()` が false を返すことに気づいた経緯。

### モックの様子がおかしい

テストコードで Memcached クラスのモックを使った時に様子がおかしかった。

```
class HogeTest extends PHPUnit\Framework\TestCase
{
    public function testHoge()
    {
        $m = $this->getMockBuilder('\Memcached')
            ->setMethods(['getStats'])
            ->getMock();
        $m->method('getStats')
          ->willReturn(false);

        $m->getStats();
    }
}
```

これを実行すると、引数が足りないとのことで怒られてしまう。

```
PHPUnit 6.1.1 by Sebastian Bergmann and contributors.

There was 1 error:

1) HogeTest::testHoge
ArgumentCountError: Too few arguments to function Mock_Memcached_9d6a5f24::getStats(), 0 passed in /Users/akihito1/dev/HogeTest.php on line 19 and exactly 1 expected
```

プロダクトコードでは実際に引数なしで動いてるのに...おかしいなぁと思い、PHPUnit のコードを追っていった。

### 生成されたモックのソースコードがおかしい → memcached 拡張の問題だった

モックの機能は [phpunit-mock-objects](https://github.com/sebastianbergmann/phpunit-mock-objects) に切り出されていて、PHPUnit はこれを利用している。

`getMockBuilder()` を使ったとき、対象のクラスを継承するソースコードを生成していて、今回でいうと下記のようなコードになる。  
元クラスのメソッド呼び出しを `InvocationMocker` が仲介することで、予め定義したマッチャの条件を満たしているかをテストできるようになっている。

```php
class Mock_Memcached_8ccc9d9d extends Memcached implements PHPUnit_Framework_MockObject_MockObject
{
    // ...

    public function getStats($args)
    {
        $arguments = array($args);
        $count     = func_num_args();

        if ($count > 1) {
            $_arguments = func_get_args();

            for ($i = 1; $i < $count; $i++) {
                $arguments[] = $_arguments[$i];
            }
        }

        $result = $this->__phpunit_getInvocationMocker()->invoke(
            new PHPUnit_Framework_MockObject_Invocation_Object(
                'Memcached', 'getStats', $arguments, '', $this, false
            )
        );

        return $result;
    }
    
    // ...
}

```

今回問題なのは、生成された `getStats()` の引数が省略可能になっていないので `ArgumentCountError` が起きてしまっていること。  
モックの生成処理を追っていったら、引数は [ReflectionParameter::isOptional() で判定している](https://github.com/sebastianbergmann/phpunit-mock-objects/blob/3819745c44f3aff9518fd655f320c4535d541af7/src/Generator.php#L1116-L1117) ことがわかった。

この判定が false になっている = memcached 拡張の方に問題があることがわかったので調べたら、ドキュメントに載ってない引数があることがわかって......という流れで「やったこと」につながる。


## まとめ

結果的にはたった 4行 のちょっとした修正だけど、そこに至るまでにいろいろと調べていたので書き留めておいた。

普段は（時間的な制約があるなかで開発してる場合は特に）場当たり的な解決策に走ってしまったり、余裕があったら調べようと思って後回しにしてそのまま揮発してしまうことがあるが、そういった誘惑に負けずに腰を据えてじっくり調べることで、より理解が深まったりオープンソースに貢献できることを体験できた。

コードの中に深く潜っていくのは楽しくてワクワクする。  
今回のような経験を積み重ねていけば、より深く潜れるようになっていくのかもしれない。