+++
date = "2014-02-01T16:25:32+09:00"
draft = false
title = "phpでバイナリ / テキストファイルの判定"
tags = ["php"]
+++

拡張子での判定は、除外対象のメンテが必要になったりするので今回はボツです。

最良の方法か分かりませんが、ファイル内にnull文字が含まれる場合にバイナリファイルとして判定するようにしました。

<!--more-->

```php
<?php
$result = preg_match('#\0#', file_get_contents($file));

if ($result === 1) {
    echo 'binary';
} elseif ($result === 0) {
    echo 'text';
}
```
より良い方法がありましたらご教授ください m(＿ ＿)m

#### 2014-04-06 追記
はてブでコメントいただいた方法を試しました。

- NULLバイトを含むかの判定だけなので strpos で事足りる。
- 対象ファイルのサイズが大きいと色々と困るので stream を使う。

```php
<?php
function is_binary($file) {
    $fp = fopen($file, 'r');
    while ($line = fgets($fp)) {
        if (strpos($line, '\0') !== false) {
            fclose($fp);
            return true;
        }
    }

    fclose($fp);
    return false;
}
```

900MB のバイナリファイルを使って、最初のスクリプトと実行時間を比較しました。

##### 最初のスクリプト
```
$ time php 1.php
php 1.php  0.37s user 2.09s system 6% cpu 35.830 total
```

##### 改善後
```
$ time php 2.php
php 2.php  0.04s user 0.02s system 49% cpu 0.116 total
```

全然違う…！(・∀・)  
最初のなんか、時間を測る以前に Allowed memory size  
エラーが出ちゃいました。。

大変勉強になりました m(＿ ＿)m
