+++
date = "2012-08-28T16:18:32+09:00"
draft = false
title = "HerokuのWebサーバーをUnicornに変更する"
tags = ["ruby", "heroku"]
+++

最近PHPネタばかりだったので、頑張ってRailsについて書いてみます。  
RailsではデフォルトでWEBrickが起動しますが、低速なので本番運用には向かないとされています。

<!--more-->

<a href="http://www.amazon.co.jp/WEB-DB-PRESS-Vol-70-%E6%88%90%E7%94%B0/dp/4774151904"
target="_blank">WEB+DB PRESS Vol.70</a>

![WEB+DB PRESS
Vol.70](https://dl.dropbox.com/u/22083548/octopress/20120827/webdb_vol70.jpeg)


WEB+DB PRESS vol.70でRails高速化としてUnicornが紹介されています。  
普段Railsで開発するときはherokuを使っているので  
herokuでUnicornを使ってみたいと思います。

#### heroku ps を確認
まずはherokuで現在使われているWebサーバーを確認します。

```
$ heroku ps
```
![heroku_ps](https://dl.dropbox.com/u/22083548/octopress/20120827/heroku_ps_thin.png)


herokuのデフォルトはthinなのでしょうか？？  
以下、Unicornのインストールを進めていきます。

#### Gemfileに追加
```
gem 'unicorn'
```
#### config/unicorn.rbを作成
とりあえず設定内容は下記にしました。  
詳しいことは勉強中です。すみません。  

```
worker_processes 2
timeout 20
preload_app false
stdout_path "log/unicorn-out.log"
stderr_path "log/unicorn-err.log"
```

#### Procfileを作成
Railsのルートディレクトリ直下にProcfileを作成します。

```
web: bundle exec unicorn -p $PORT -c ./config/unicorn.rb  
```

#### herokuにpush
いつものようにherokuにpushします。

```
$ git push heroku master
```

#### heroku ps で確認
```
$ heroku ps
```
![heroku_ps_unicorn](https://dl.dropbox.com/u/22083548/octopress/20120827/heroku_ps_unicorn.png)


bundle exec unicorn …となっていれば成功です。
heroku psの出力の2行目が

```
web.1: crashed for…
```
になっていたら設定を見なおしてみてください。
