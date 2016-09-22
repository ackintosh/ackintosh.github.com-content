+++
date = "2013-08-05T17:42:28+09:00"
draft = false
title = "PHPで関数合成を書いてみる"
tags = ["php"]
+++

<a href="http://qiita.com/yuya_takeyama/items/858c5a0652441f54f0a8"
target="_blank">PHP で関数合成 - Qiita [キータ]</a>

こちらの投稿がとても興味深かったので、自分なりに書いてみました。

<!--more-->

```php
<?php
class Compose
{
    public static function _()
    {
        $callables = func_get_args();

        return array_reduce($callables, function ($a, $b) {
            if ($a === null) {
                return function () use ($b) {
                    return call_user_func_array($b, func_get_args());
                };
            } else {
                return function ($arg) use ($a, $b) {
                    return call_user_func($b, $a($arg));
                };
            }
        });

    }
}


$mapUcfirst = function ($arg) {
    return array_map('ucfirst', $arg);
};

$splitWithUnderscore = function ($arg) {
    return explode('_', $arg);
};


$camelize = Compose::_($splitWithUnderscore, $mapUcfirst, 'join');
var_dump($camelize('man_with_a_mission'));
// ManWithAMission
```
