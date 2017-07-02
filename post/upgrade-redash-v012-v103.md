+++
date = "2017-07-02T12:42:45+09:00"
draft = false
title = "Docker で動かしてる Redash を v0.12 から v1.0.3 にアップグレード"
+++

職場で使ってる Redash をアップグレードしたかったので、ローカルで素振りしたときのメモ。

<!--more-->

[How to Upgrade Redash · Redash Help Center](https://redash.io/help-onpremise/maintenance/how-to-upgrade-redash.html) のとおり、基本的にはコマンド一発で終わるけど、Docker で動かしてる場合は「コンテナにログインしてコマンド叩くとか NG だよな...」という疑問がわいてくるので、同じ状況で悩んでるかたの役に立ったら幸いです。

## コンテナをとめる

```
$ docker-compose down
```

## DB のバックアップをとる

Data Volume でデータをホストマシンと共有していたので単純に tar で固めた。

```
$ tar czvf postgres-data.bk.tar.gz postgres-data
```


## compose ファイルを更新

v1 では内容がガラッと変わってるので、それに合わせてファイルを更新した。

[v1.0.3/docker-compose.production.yml](https://github.com/getredash/redash/blob/v1.0.3/docker-compose.production.yml)

使いたいイメージのバージョンを指定しておいた。

```
  server:
    image: redash/redash:1.0.3.b2896
...    
  worker:
    image: redash/redash:1.0.3.b2896
```

v1 では PostgreSQL 9.5 になっているが、旧 compose ファイルで指定されてた 9.3 にしておいた。

```
  postgres:
    image: postgres:9.3
```


## DB マイグレーションを実行

```
$ docker-compose run --rm server manage db upgrade

WARNING: Found orphan containers (redash_redash-nginx_1, redash_redash_1) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
Starting redash_redis_1 ...
Starting redash_postgres_1 ... done
[2017-07-02 03:28:39,179][PID:1][INFO][root] Generating grammar tables from /usr/lib/python2.7/lib2to3/Grammar.txt
[2017-07-02 03:28:39,202][PID:1][INFO][root] Generating grammar tables from /usr/lib/python2.7/lib2to3/PatternGrammar.txt
[2017-07-02 03:28:40,139][PID:1][INFO][alembic.runtime.migration] Context impl PostgresqlImpl.
[2017-07-02 03:28:40,140][PID:1][INFO][alembic.runtime.migration] Will assume transactional DDL.
[2017-07-02 03:28:40,266][PID:1][INFO][alembic.runtime.migration] Running upgrade  -> 65fc9ede4746, Add is_draft status to queries and dashboards
[2017-07-02 03:28:40,353][PID:1][INFO][alembic.runtime.migration] Running upgrade 65fc9ede4746 -> d1eae8b9893e, add Query.schedule_failures
```


## 起動

旧 compose ファイルで定義していたコンテナを削除するために `--remove-orphans` を指定する。

```
$ docker-compose up --remove-orphans

Removing orphan container "redash_redash-nginx_1"
Removing orphan container "redash_redash_1"
redash_postgres_1 is up-to-date
redash_redis_1 is up-to-date
Creating redash_worker_1 ...
Creating redash_server_1 ...
Creating redash_smtp_1 ...
Creating redash_worker_1
Creating redash_smtp_1
Creating redash_server_1 ... done
Creating redash_smtp_1 ... done
Creating redash_nginx_1 ... done
Attaching to redash_postgres_1, redash_redis_1, redash_worker_1, redash_server_1, redash_smtp_1, redash_nginx_1
```

アップグレードできた！

![image](https://user-images.githubusercontent.com/1885716/27767158-2dd05d2a-5f28-11e7-9311-f46f2f39ed66.png)

(このあとお好みで PostgreSQL を 9.3 -> 9.5 にアップグレード)


## (補足) コンテナでアップグレードスクリプトを実行してもエラーになる

めんどくさいからコンテナにログインしてアップグレードしちゃえ！と思っても、下記のエラーが出てアップグレードできない 👍


##### アップグレードスクリプトをダウンロード

```
$ docker-compose run redash bash
# wget https://raw.githubusercontent.com/getredash/redash/master/bin/upgrade
# chmod +x upgrade
```

##### エラー

```
# ./upgrade

Starting Redash upgrade:
/usr/local/lib/python2.7/dist-packages/requests/packages/urllib3/util/ssl_.py:318: SNIMissingWarning: An HTTPS request has been made, but the SNI (Subject Name Indication) extension to TLS is not available on this platform. This may cause the server to present an incorrect TLS certificate, which can cause validation failures. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/security.html#snimissingwarning.
  SNIMissingWarning
/usr/local/lib/python2.7/dist-packages/requests/packages/urllib3/util/ssl_.py:122: InsecurePlatformWarning: A true SSLContext object is not available. This prevents urllib3 from configuring SSL appropriately and may cause certain SSL connections to fail. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/security.html#insecureplatformwarning.
  InsecurePlatformWarning
Found version: 1.0.3
Current version: current
Traceback (most recent call last):
  File "./upgrade", line 236, in <module>
    deploy_release(args.channel)
  File "./upgrade", line 214, in deploy_release
    verify_minimum_version()
  File "./upgrade", line 186, in verify_minimum_version
    if semver.compare(current_version(), '0.12.0') < 0:
  File "/usr/local/lib/python2.7/dist-packages/semver.py", line 54, in compare
    v1, v2 = parse(ver1), parse(ver2)
  File "/usr/local/lib/python2.7/dist-packages/semver.py", line 21, in parse
    raise ValueError('%s is not valid SemVer string' % version)
ValueError: current is not valid SemVer string
```

> Current version: current  
> ValueError: current is not valid SemVer string

アップグレードスクリプトは `current` -> `バージョン番号のディレクトリ` にリンクされている前提で書かれているが、Docker イメージはそうなっていない。  
current(実ディレクトリ)の中に Redash がインストールされているので、現在のバージョンが判定できずエラーになっている 👍
