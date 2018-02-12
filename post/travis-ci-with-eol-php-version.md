---
title: "Travis CIでPHP5.3から最新までのバージョンでCIをまわす"
date: 2018-02-12T16:05:25+09:00
draft: false
tags: ["php"]
---

なんだか辛さが滲み出るようなタイトルだ。どうしても古いPHPのサポートを継続しておきたいライブラリがあって、PHP5.3から最新までのバージョンでCIをまわすときに少しハマりどころ(といったら大げさだが)があったのでメモしておく。

<!--more-->

## どんなライブラリか

いきなり話が脇道にそれるが、"どうしても古いPHPのサポートを継続しておきたいライブラリ"とは何かというと [Toumi](https://github.com/ackintosh/toumi) という、phpファイルのクラスや関数宣言の部分だけを抜き出してincludeすることが出来るライブラリ。

[クラスや関数宣言だけをインクルードできるライブラリを作りました - 暁](/blog/2013/11/24/toumi/)

> クラスや関数の宣言と諸々の処理がごちゃ混ぜに書かれてるスクリプトをメンテナンスする時、
> リファクタリングするためにテストを書きたいけど、テストを書くためにはリファクタリングしないと…(＊_＊) という状況ありませんか？

つまり、超レガシーなコードをテストで保護するためのはじめの一歩としてとても重宝するライブラリ。なので、そもそもToumiが古いPHPのバージョンをサポートしなくなってしまったら、これを必要としている人たちが使えなくなってしまうかもしれない。というのがサポートを継続しておきたい理由。

## PHP5.3はTrustyではサポートされていない

[The Trusty Build Environment - Travis CI](https://docs.travis-ci.com/user/reference/trusty#PHP-images)

> Note: We do not support PHP versions 5.2.x and 5.3.x on Trusty. Specifying it will result in build failure. If you need to test with these versions, use Precise.

Travis CIでのユニットテストはデフォルトではUbuntuのTrustyで実行されるが、これが5.3をサポートしていないので .travis.yml でPreciseを明示する必要がある。

ちなみに、Preciseが明示されていない場合は下記のようなエラーになる。

[Job #17.1 - ackintosh/toumi - Travis CI](https://travis-ci.org/ackintosh/toumi/jobs/340079648)

> PHP 5.3 is supported only on Precise.  
> See https://docs.travis-ci.com/user/reference/trusty#PHP-images on how to test PHP 5.3 on Precise.
Terminating.

## PHPUnitは4系を使う

PHPUnitは5系からはphp5.6以降のサポートなので、4系を使わなければならない。

## おわりに

本当にメモ程度で中身が薄いが何事もアウトプットだと思って、週末の盆栽作業の内容を書いた。
