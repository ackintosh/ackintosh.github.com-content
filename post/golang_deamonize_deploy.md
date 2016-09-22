+++
date = "2016-08-17T12:40:37+09:00"
draft = false
title = "Goアプリのデーモン化とデプロイの仕組み"
+++

社内の某合宿イベントで、Go製の軽量WAF [Echo](https://echo.labstack.com/) を使ったAPIサーバーを作ろうとしていて、夏休み中にデーモン化とデプロイの仕組みを作ってみたので、ちょっとまとまりきってないですが忘れないうちにメモしておきます。

<!--more-->

慣れない事が多くて試行錯誤しながら丸一日使ってめっちゃ疲れたけど勉強になった。hot deploy の仕組みが大変興味深いです。（[参考記事](#参考記事:c89a01ea34ec3d3b965d2855f1d3c3d0)）

試行錯誤した結果、利用するツール・ライブラリは下記になりました。

- デーモン化
  - [supervisord](http://supervisord.org/)

- デプロイ  
（githubに push したら アプリケーションサーバーが webhook 通知を受信してビルド・graceful restart する）
  - [facebookgo/grace](https://github.com/facebookgo/grace)
  - [mattn/gost](https://github.com/mattn/gost)


(デプロイの図)
![deploy](/images/golang_deamonize_deploy.png)



##### 試行錯誤

試行錯誤や調査の結果、利用を見送ったもの。

daemontools (プロセス管理)

- インストールが難しそう
- 開発がアクティブじゃない
- パッチを当てないといけない様子

　→ supervisord の方がインストール楽だし、設定も簡単

go-server-starter (ホットデプロイ)

- graceful shutdown に対応していない
- graceful shutdown に対応するために manners を使おうとするとアプリケーションのソースコードに手をいれないといけない
  - Echo のレールから外れてしまう

　→ grace なら最小限の変更で済む。公式に[サンプルコード](https://echo.labstack.com/recipes/graceful-shutdown)がある。

##### 展望

- ビルドサーバーを用意してバイナリを配布する仕組みを作ってみたい
- [monochromegane/torokko](https://github.com/monochromegane/torokko) が気になる。
  - [Goのデプロイを「もっと」簡単にする。ビルドプロキシCargo。改めTorokko。](http://blog.monochromegane.com/blog/2015/08/16/deploy-golang-by-cargo/)


##### 参考記事

- [Golangを初めて本番投入したぜ！](http://blog.yusuke.be/entry/2016/01/18/111838)
- [Go言語でGraceful Restartをする](https://shogo82148.github.io/blog/2015/05/03/golang-graceful-restart/)
- [Server::Starterから学ぶhot deployの仕組み](http://blog.shibayu36.org/entry/2012/05/07/201556)
- [「Server::Starterに対応するとはどういうことか」の補足](http://blog.livedoor.jp/sonots/archives/40248661.html)
- [Go言語でWebAppの運用に必要なN個のこと](http://mattn.kaoriya.net/software/lang/go/20130918122901.htm)
