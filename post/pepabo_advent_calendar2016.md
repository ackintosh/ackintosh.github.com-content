+++
date = "2016-12-17T21:24:33+09:00"
draft = false
title = "充実した1年のスタートを ― PHPで"

+++

この記事は [pepabo Advent Calendar 2016](http://qiita.com/advent-calendar/2016/pepabo) 17日目の記事です。

<!--more-->

#### ご存知でしょうか？

珍しい組み込み関数があることで有名なPHP[要出典]ですが、 **date_sunrise** という関数をご存知でしょうか。

[PHP: date_sunrise - Manual](http://php.net/manual/ja/function.date-sunrise.php)

> date_sunrise — 指定した日付と場所についての日の出時刻を返す

試しに、ペパボ本社がある東京都渋谷区セルリアンタワーの元旦の日の出を出力してみます。

```sh
$ php -d date.timezone='Asia/Tokyo' -r 'echo date_sunrise(mktime(0, 0, 0, 1, 1, 2017), SUNFUNCS_RET_STRING, 35.39, 139.41);'

06:51
```



素晴らしい...。  
これを使って色々やれば初日の出を寝過ごすことなく、充実した1年をスタートさせることができます。必要なのは PHP とその他のごくわずかな手間だけです。


ただ、 **この時間が本当にあってるのかちょっと不安** です。

#### ソースコード探索

ということで date_sunrise の実装を覗いてみましょう。 (PHP7.1.0)

[php/php-src: The PHP Interpreter](https://github.com/php/php-src/tree/PHP-7.1)


関数は `PHP_FUNCTION` マクロを使って定義されているので `PHP_FUNCTION(date_sunrise)` で検索すればすぐに目的のコードにたどり着けます。

https://github.com/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c#L4834


<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c?slice=4830:4846"></script>

内部的には日の入り時間を返す [date_sunset](http://php.net/manual/ja/function.date-sunset.php) とかなり共通していて、`php_do_date_sunrise_sunset` の第2引数に渡すフラグで処理を分けているようです。

[INTERNAL_FUNCTION_PARAM_PASSTHRU マクロ](https://github.com/php/php-src/blob/PHP-7.1.0/Zend/zend.h#L49)は、その名の通り引数を利用しない場合のアテの役割で、 `execute_data, return_value` に展開されます。

なぜアテが必要かというと、 `PHP_FUNCTION(date_sunrise)` はコンパイラによって最終的に `zif_date_sunrise(zend_execute_data *execute_data, zval *return_value)` に展開されるので、この第1, 2引数を埋めるために INTERNAL_FUNCTION_PARAM_PASSTHRU マクロを使っています。


つづいて `php_do_date_sunrise_sunset` の中を順番に見ていきます。

<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c?slice=4743:4762"></script>

変数の初期化と引数を解析しています。

`zend_parse_parameters` は第1引数で実行時に渡された引数の数、第2引数で引数の情報を受取り、解析した結果を第3引数以降に代入します。  
また、第2引数 `"l|ldddd"` は、各引数の型と必須・オプションを表していて、パイプ `|` で区切られた後ろの2つめ以降がオプショナルであることがわかります。

<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c?slice=4761:4783"></script>

つづいて、実行時に渡された引数の数によって、デフォルト値を設定しています。switch 文のフォールスルーがうまく使われていて勉強になります。  
`INI_FLT` は php.ini の設定値を取得するための double 型用のマクロです。[マニュアル](http://php.net/manual/ja/function.date-sunrise.php)を見ると、たしかに `ini_get("date.default_latitude")` 等がデフォルトで使われると書かれています。

<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c?slice=4783:4790"></script>

第2引数で渡す format が、予め定義されているどれにも該当しない場合に警告を出力しています。PHP で開発しているとよく見るあの警告は `php_error_docref` で出力されているようです。

<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/php_date.c?slice=4791:4805"></script>

ようやく時間を計算してそうなコードまでたどり着きました !!  
肝心の計算をしているであろう `timelib_astro_rise_set_altitude` の中を見てみます。

<script src="http://gist-it.appspot.com/github/php/php-src/blob/PHP-7.1.0/ext/date/lib/astro.c?slice=211:296"></script>


フムフム...

...

...

**なんかちゃんと計算してることがわかりました !**

#### 充実した1年のスタートを ― PHPで

**緻密に計算された日の出時刻を返してくれる** date_sunrise 関数。

繰り返しになりますが、これを使って色々やれば初日の出を寝過ごすことなく、充実した1年をスタートさせることができます。必要なのは PHP とその他のごくわずかな手間だけです。


