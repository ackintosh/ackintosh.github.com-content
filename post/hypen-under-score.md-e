+++
date = "2013-08-11T16:10:19+09:00"
draft = false
title = "ハイフンとアンダースコアの使い分け"
tags = ["ruby"]
+++

ネーミングの時のハイフンとアンダースコアの使い分けが、自分の中で曖昧なところがあったのでメモ。

言語やフレームワークによって色々あるかもしれませんが、以下、Ruby(gem)の場合です。

#### Eric Hodel氏の推奨するネーミングルール
RubyGemsの作者、Eric Hodel氏は自身のブログで次のように推奨しています。

<!--more-->

<a href="http://blog.segment7.net/2010/11/15/how-to-name-gems"
target="_blank">How to Name Gems</a>

> Here is my STRONG recommendation on how to name gems:
> Use underscores
> ・fancy_require
> ・newrelic_rpm
> ・ruby_parser
> This matches the file the user will require and makes it easier for the
> user to start using your gem. gem install my_gem will be loaded by
> require ‘my_gem’.
> Use dashes for extensions
> ・net-http-persistent
> ・rdoc-chm
> ・autotest-growl
> If you’re adding functionality to another gem use a dash. The dash is
> different-enough from an underscore to be noticeable. If you tilt the
> dash a bit it becomes a slash as well, making it easier for the user to
> know what to require. gem install net-http-persistent becomes require
> ‘net/http/persistent’
要するに

ハイフン -> パスの区切り  
アンダースコア -> 単語の区切り  
といったところでしょうか。  

#### 試してみる
##### ハイフン区切り

```
$ bundle gem ackintosh-tiny-progressbar
```

````
module Ackintosh
  module Tiny
    module Progressbar
      # Your code goes here ...
    end
  end
end
````
全て別のモジュールに分かれています。

##### アンダースコア区切り

````
$ bundle gem ackintosh-tiny_progressbar
````

````
module Ackintosh
  module TinyProgressbar
    # Your code goes here…
  end
end
````
「tiny」と「progressbar」は別の単語ですが意味的には１つになっています。
