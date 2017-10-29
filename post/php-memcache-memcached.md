---
title: "PHP の Memcache と Memcached は相互に読み書きできない"
date: 2017-10-16T21:13:17+09:00
draft: false
tags: ["php"]
---

PHP から memcached を利用するための拡張モジュールには2種類ある。

#### Memcache

- [PHP: Memcache - Manual](http://php.net/manual/ja/book.memcache.php)
- [72.52.91.13 Git - pecl/caching/memcache.git/summary](http://git.php.net/?p=pecl/caching/memcache.git)


#### Memcached

- [PHP: Memcached - Manual](http://php.net/manual/ja/book.memcached.php)
- [php-memcached-dev/php-memcached: memcached extension based on libmemcached library](https://github.com/php-memcached-dev/php-memcached/)

<!--more-->

Memcache は開発が止まっている。Memcached は libmemcached を使って実装され、現在も開発が続けられていてもちろん PHP7 にも対応している。ということで新規なら迷わず **Memcached** を選択する。

しかし歴史のある Web アプリケーションでは、(おそらく)開発当時は Memcached が無かったので Memcache が使われ今も動き続けている。そして PHP7 へのバージョンアップのために Memcached への移行という課題を抱えているエンジニアもいらっしゃるのではないか。（私もそのひとり）


記事タイトルのとおり、「Memcache と Memcached は相互に読み書きできない」。つまり Memcache で書き込んだデータは Memcached では読めないし、その逆もまた然り。  
例えば memcached を介して複数のロールでデータを共有している場合は、関係するロール全てを同時に Memcached に移行する必要がある。

「`set()` と `get()` しか使ってないし、Memcache/Memcached どっちも同じこと（memcached にデータを読み書き）してるだけなんだからシュッといけるでしょ〜」と楽観しているかたは当記事を読んで現実を受け入れ、考えを改める必要がある。  
※ 単一のロールに閉じて memcached を使っている場合は上記のような大変さは無い。ただ、Memcache/Memcached の間でデータの読み書きができないので、移行したタイミングでデータが全て無かったことになる、または意図しないデータが読み出される(後述)可能性は考慮しておいた方が良さそう。

以下、Memcache/Memcached でデータを読み書きした場合の挙動の確認と、それぞれの拡張の実装の違いを見ていく。

## Memcache同士、Memcached同士で読み書き

（※ TODO: 各拡張のバージョン等を追記する）  
まずは通常のケース。

### Memcache で set → Memcache で get

- set.php
```php
<?php
$m = new Memcache();
$m->addServer('127.0.0.1', 11211);
$m->set('key_string', 'value_string');
$m->set('key_bool', true);
```

- get.php
```php
<?php
$m = new Memcache();
$m->addServer('127.0.0.1', 11211);
var_dump($m->get('key_string'));
var_dump($m->get('key_bool'));
```

- 結果
```
[vagrant@local memcache]$ php set.php
[vagrant@local memcache]$ php get.php
string(15) "value_string"
string(1) "1"
```

`key_bool ` にセットした `true` が `"1"` に変わっている (！)


### Memcached で set → Memcached で get

- set.php
```php
<?php
$m = new Memcached();
$m->addServer('127.0.0.1', 11211);
$m->set('key_string', 'value_string');
$m->set('key_bool', true);
```

- get.php
```php
<?php
$m = new Memcached();
$m->addServer('127.0.0.1', 11211);
var_dump($m->get('key_string'));
var_dump($m->get('key_bool'));
```

- 結果
```
[vagrant@local memcached]$ php set.php
[vagrant@local memcached]$ php get.php
string(15) "value_string"
bool(true)
```

期待通り :ok:


## "データを読み書きできない" の具体的な挙動を確認する

### Memcached で set → Memcache で get

( 逆パターン(Memcache → Memcached)は試していない )

- set.php
```php
<?php
$m = new Memcached();
$m->addServer('127.0.0.1', 11211);
$m->set('key_string', 'value_string');
$m->set('key_bool', true);
```

- get.php
```php
<?php
$m = new Memcache();
$m->addServer('127.0.0.1', 11211);
var_dump($m->get('key_string'));
var_dump($m->get('key_bool'));
```

- 結果
```
[vagrant@local memcached_memcache]$ php set.php
[vagrant@local memcached_memcache]$ php get.php
string(15) "value_string"
bool(false)
```

`$m->get('key_bool')` の結果が false → おそらく「取得に失敗した」意味の false

##### 色々な型で試す

- set.php
```php
<?php
$m = new Memcached();
$m->addServer('127.0.0.1', 11211);
$m->set('key_string', 'value_string');
$m->set('key_bool', true);
$m->set('key_int', 1);
$m->set('key_array', array('value_array'));
```

- get.php
```php
<?php
$m = new Memcache();
$m->addServer('127.0.0.1', 11211);
var_dump($m->get('key_string'));
var_dump($m->get('key_bool'));
var_dump($m->get('key_int'));
var_dump($m->get('key_array'));
```

- 結果
```
[vagrant@local memcached_memcache]$ php get.php
string(12) "value_string"
bool(false)
bool(false)
bool(false)
```

　→ string 以外が全部 false...

##### 配列が unserialize されない

- set.php
```php
<?php
$m = new Memcached();
$m->addServer('127.0.0.1', 11211);
$m->set('key_array', array('value_array'));
```

- get.php
```php
<?php
$m = new Memcache();
$m->addServer('127.0.0.1', 11211);
var_dump($m->get('key_array'));
```

- 結果
```
[vagrant@local memcached_memcache]$ php get.php
string(29) "a:1:{i:0;s:11:"value_array";}"
```



## 実装の違いを確認する

ここまで見てきた挙動の要因になっている Memcache/Memcached それぞれの `set()` `get()` の実装の違いを確認する。

### Memcache


#### set()

[ソースコード](http://git.php.net/?p=pecl/caching/memcache.git;a=blob;f=memcache.c;h=6a7576d1a86fdbeaf8f30c3cfd54ac642719c006;hb=HEAD#l1820)

###### 値の保存の仕方

- string: そのまま保存
- int, float, bool: 文字列に変換して保存
- それ以外: シリアライズして保存
  - 保存時に、シリアライズしたかどうかを `flags` で渡す
    - http://git.php.net/?p=pecl/caching/memcache.git;a=blob;f=memcache.c;h=6a7576d1a86fdbeaf8f30c3cfd54ac642719c006;hb=HEAD#l1864

#### get()

[ソースコード](http://git.php.net/?p=pecl/caching/memcache.git;a=blob;f=memcache.c;h=6a7576d1a86fdbeaf8f30c3cfd54ac642719c006;hb=HEAD#l1435)

- 保存時にシリアライズしていれば [アンシリアライズする](http://git.php.net/?p=pecl/caching/memcache.git;a=blob;f=memcache.c;h=6a7576d1a86fdbeaf8f30c3cfd54ac642719c006;hb=HEAD#l1305)

### Memcached

#### set()

[ソースコード](https://github.com/php-memcached-dev/php-memcached/blob/r2.1.0/php_memcached.c#L1107
)


###### 値の保存の仕方

- string, int, float, bool: そのまま(シリアライズせず)保存
- それ以外: シリアライズして保存
- 保存時に、型の情報を `flags` で渡す
  - https://github.com/php-memcached-dev/php-memcached/blob/r2.1.0/php_memcached.c#L1422
  - 第7引数 `flags` はライブラリが自由に利用できるフィールドで、Memcached はここに型情報を入れている
  - http://docs.libmemcached.org/memcached_set.html

#### get()

[ソースコード](https://github.com/php-memcached-dev/php-memcached/blob/r2.1.0/php_memcached.c#L490)

- [`flags` に保存しておいた型情報を元に返り値をいい感じにする](https://github.com/php-memcached-dev/php-memcached/blob/r2.1.0/php_memcached.c#L2932)


## まとめ

当初は全く理解できなかった謎の挙動だが、実装の違いがわかると腑に落ちてくるのではないだろうか。

シュッと移行できるだろうとかなり楽観していたのだが辛い現実を目の当たりにしてしまい、その勢いでブログに書いた。そのため必要以上に不安を煽るような書き方をしてしまった箇所があるかもしれない。

...

複数のロールで memcached を介してデータを共有している場合でも、うまいこと1ロールずつ移行する方法がないものか。"こうやって移行したぜ！" という知見をお持ちの方は [@NAKANO_Akihito](https://twitter.com/NAKANO_Akihito/) までご連絡いただけると幸いです。

(追記)  
Memcache をフォークして PHP7 対応してるリポジトリがあると知ったのでブログに残した。  
[PHP7 に対応した Memcache 拡張モジュール](/blog/2017/10/27/pecl-memcache/)