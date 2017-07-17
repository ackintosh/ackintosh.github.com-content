+++
date = "2017-07-11T01:43:26+09:00"
draft = false
title = "Cowrie on Docker"

+++

[micheloosterhof/cowrie: Cowrie SSH/Telnet Honeypot](https://github.com/micheloosterhof/cowrie)

<!--more-->

SSH/Telnet のハニーポット。[Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%8F%E3%83%8B%E3%83%BC%E3%83%9D%E3%83%83%E3%83%88)に載っている種類でいうと低対話型ハニーポットになると思う。  
（あってるかわからないけど カウリー と読んでる）  
今回は、どんなログが取れるのかを見てみたかっただけなので "動かしてみた" 程度の内容だが、`/etc/passwd` などをエミュレートしたり、攻撃者が仕掛けたマルウェアを特定のディレクトリに隔離しておく機能があるようなので面白そう。

（本当は T-Pot を試してみたかったけど、要件を満たすインスタンスを用意するのが金銭的にアレだったので、とりあえず (T-Pot の構成要素のひとつである) Cowrie だけ）

- T-Pot
  - [dtag-dev-sec/tpotce: T-Pot Image Creator](https://github.com/dtag-dev-sec/tpotce#concept)
  - [EC2上にHoneypot(T-Pot)をインストールして、サイバー攻撃をELKで可視化してみた - Qiita](http://qiita.com/tarosaiba/items/871ab0c155578f8a38fe)



以下、ubuntu16.04 の Docker コンテナで Cowrie を動かすまでの手順。

##### Docker をインストール


Docker Community Edition をインストール  
[Get Docker CE for Ubuntu | Docker Documentation](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

Docker Compose をインストール  
[Install Docker Compose | Docker Documentation](https://docs.docker.com/compose/install/)

##### cowrie 作者が用意してくれている Docker イメージを使う

[micheloosterhof/docker-cowrie: Docker Cowrie Honeypot image](https://github.com/micheloosterhof/docker-cowrie)


```
$ git clone https://github.com/micheloosterhof/docker-cowrie.git && cd docker-cowrie
```

##### micheloosterhof/cowrie のイメージは無いのでビルドする

docker-compose.yml

```diff
     # Bug Michel to create a Docker Hub account and build an image from the repo.
-    # build: .
-    image: micheloosterhof/cowrie:dev
+    build: .
+    #image: micheloosterhof/cowrie:dev
```

##### cowrie のログを見れるように Data Volume でホストマシンと共有する

```
$ mkdir log
```

docker-compose.yml

```diff
volumes:
      # Map a local cowrie config dir into the container.
      # - "./etc:/cowrie/cowrie-git/etc"
      # Make cowrie logs persistent through container recreates.
      - "./var:/cowrie/cowrie-git/var"
+     - "./log:/cowrie/cowrie-git/log"
```

##### 起動して放置してると...

```
$ docker-compose up -d
```
攻撃キタ!!

```
$ tail -f log/cowrie.json
...
...

{"eventid": "cowrie.login.failed", "username": "jenkins", "timestamp": "2017-07-10T13:40:53.763171Z", "message": "login attempt [jenkins/1234] failed", "syste
m": "SSHService 'ssh-userauth' on HoneyPotSSHTransport,19,52.88.114.5", "isError": 0, "src_ip": "52.88.114.5", "session": "fa045b057360", "password": "1234",
"sensor": "e4bba5c4b1e1"}

```