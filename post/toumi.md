+++
date = "2013-11-24T15:35:00+09:00"
draft = false
title = "クラス / 関数宣言だけをインクルードできるライブラリを作りました"
tags = ["php", "test"]
+++


クラスや関数の宣言と諸々の処理がごちゃ混ぜに書かれてるスクリプトをメンテナンスする時、  
リファクタリングするためにテストを書きたいけど、テストを書くためにはリファクタリングしないと…(＊_＊) という状況ありませんか？

<!--more-->

例えば

```php
<?php
require_once 'xxx.php';

function hoge($arg)
{
    return 'hoge' . $arg;
}

somefunction(1234);

set('hoge', hoge('fuga'));
render('hoge.html');
exit;
```

こんな感じのコードがあって、hoge()関数のテストを書きたい時
関数宣言の部分だけインクルードできれば、既存コードに一切手を入れずにテスト書き始められます。

ということで作りました。

<a href="https://github.com/ackintosh/toumi" target="_blank">ackintosh /
toumi</a>

このライブラリを使って上記スクリプトをインクルードすれば、
下記のようにテストが書けます。

```php
<?php
// Only the function declaration is included.
Ackintosh_Toumi::load('legacy.php');

class LegacyTest extends PHPUnit_Framework_TestCase
{
    public function test_hoge()
    {
        $this->assertSame('hogefuga', hoge('fuga'));
    }
}
```

そもそもこれを使わずに済めば良いんですが…(・_・;)
