---
title: "PHP7 に対応した Memcache 拡張モジュール"
date: 2017-10-27T21:37:22+09:00
draft: false
tags: ["php"]
---

[PHP の Memcache と Memcached は相互に読み書きできない](/blog/2017/10/16/php-memcache-memcached/) を社内 slack で共有したところ、Memcache をフォークして PHP7 に対応しているリポジトリがあることを教えてもらった。  

[websupport-sk/pecl-memcache](https://github.com/websupport-sk/pecl-memcache)

<!--more-->

また、上記リポジトリと同じかどうか分からないが remi リポジトリに php-pecl-memcache のバージョン 3.0.9 が登録されていて、これが PHP7 で動作するとのことだった。（ PECL には [3.0.8 までしか登録されていない](https://pecl.php.net/package/memcache) )

Memcache を使っているアプリケーションがそのまま PHP7 にバージョンアップする道が見えてきたが、それでも個人的には下記の理由で **Memcached** に移行した方が良いと思っている。  
(※ 社内的には **Memcached** に移行する方向で動いている)

#### プロダクトとしての筋の良さ

**Memcached** は memcached サーバーとの通信部分に [libMemcached](http://libmemcached.org/libMemcached.html) を利用している。そのため、"PHP と memcached の橋渡し" に集中できているように感じる。  
一方、Memcache はライブラリを使わずソケット通信するプリミティブな実装になっている。

#### 延命している感

やはり Memcache は延命している感がある。（言語化できてないが、そう感じる。）
