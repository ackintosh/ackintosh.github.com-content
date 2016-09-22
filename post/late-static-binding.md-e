+++
date = "2013-08-25T17:23:38+09:00"
draft = false
title = "遅延静的束縛は何が嬉しいのか"
tags = ["php"]
+++

名前は見かけていたものの、いまいち理解していなかった。

<a href="http://php.net/manual/ja/language.oop5.late-static-bindings.php"
target="_blank">PHP: 遅延静的束縛 (Late Static Bindings) - Manual</a>  

<!--more-->

> PHP 5.3.0 以降、PHP に遅延静的束縛と呼ばれる機能が搭載されます。
> これを使用すると、静的継承のコンテキストで呼び出し元のクラスを参照できるようになります。
> より正確に言うと、遅延静的束縛は直近の “非転送コール”
> のクラス名を保存します。
> 静的メソッドの場合、これは明示的に指定されたクラス (通常は ::
> 演算子の左側に書かれたもの)
> となります。静的メソッド以外の場合は、そのオブジェクトのクラスとなります。
> “転送コール” とは、self:: や parent::、static:: による静的なコール、
> あるいはクラス階層の中での forward_static_call()
> によるコールのことです。 get_called_class()
> 関数を使うとコール元のクラス名を文字列で取得できます。 static::
> はこのクラスのスコープとなります。

#### 遅延静的束縛が無いと困るとき
<script src="https://gist.github.com/ackintosh/6332540.js?file=1.php"></script>

1,400,000が出力されるかと思いきや、0でした。  
・・・期待していたのと違う。

#### selfについて
PHP Manualにはこう書かれています。

> self:: あるいは CLASS による現在のクラスへの静的参照は、
> そのメソッドが属するクラス (つまり、 そのメソッドが定義されているクラス)
> に解決されます。

つまり上記のコードでいうと、getFormattedPriceメソッドはCarクラスで定義されているので、  
`self::$price` は常に `Car::$price` に解決されることになります。

これでは、いくらCarのサブクラスでgetFormattedPriceを呼び出しても価格の出力ができません。

#### 遅延静的束縛を使う
selfの代わりにstaticを使います。
<script src="https://gist.github.com/ackintosh/6332540.js?file=2.php"></script>

これで期待した通りの結果になりました。

#### staticについて
PHP Manualより

> 実行時に最初にコールされたクラスを参照するようにしています。

これにより、`static::$price` が `NissanNote::$price` に解決されたことになります。

#### もう少し深く
ただ、「最初にコールされたクラスを参照する」というのは少し浅い説明かもしれません。  
Manualには下記のようにも書かれています。

> より正確に言うと、遅延静的束縛は直近の “非転送コール” のクラス名を保存します。

例えば、
<script src="https://gist.github.com/ackintosh/6332540.js?file=3.php"></script>

上記コードだと、staticキーワードが「 最初にコールされたクラス 」を参照しているならば  
`NissanNote::getDescription();` メソッド呼び出しの結果、`static::$price` は
`NissanNote::$price` に解決されるはずですが  
実際には `Car::$price` に解決されています。

これは、「 遅延静的束縛は直近の “非転送コール” のクラス名を保存する 」と考えると理解できます。


つまり、  
`static::$price` の直近の非転送コールは `Car::getFormattedPrice()`  
なので  
`static::$price` は `Car::$price` に解決されたことになります。

非転送コールについては、下記リンクが大変参考になりました。

#### 参考にさせていただきました。
- <a
  href="http://d.hatena.ne.jp/maeharin/20130202/php_late_static_bindings"
target="_blank">PHPを愛する試み ～self:: parent:: static:: および遅延静的束縛～</a>
- <a href="http://www.ibm.com/developerworks/jp/opensource/library/os-php-53static/" target="_blank">PHP V5.3 で遅延静的バインディングを使ったオブジェクト指向プログラミングを活用する</a>
