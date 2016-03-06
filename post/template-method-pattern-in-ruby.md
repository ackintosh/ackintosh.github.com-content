+++
date = "2012-10-24T17:21:33+09:00"
draft = false
title = "RubyでTemplate Methodパターン"
tags = ["ruby"]
+++

Template Methodパターンは、アルゴリズムに多態性を持たせたい場合に有効。

Rubyは抽象メソッドをサポートしていないので、Reportクラスのoutput_lineメソッドでは例外を投げるようにしている。

<!--more-->

output_start メソッドや output_end メソッドのように、  
Template Methodの具象クラスによってオーバーライドできる非抽象メソッドをフックメソッド という。

<script src="https://gist.github.com/ackintosh/3945597.js"></script>

