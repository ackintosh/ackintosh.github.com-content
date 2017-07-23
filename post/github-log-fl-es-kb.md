+++
date = "2017-07-23T02:56:08+09:00"
draft = false
title = "GitHub の Issue 消化状況を Kibana で可視化する"
+++

主にお問い合わせ対応 Issue の起票・クローズ件数や、クローズまでにかかった時間を可視化したかったので素振り。  
以下、素振り内容をザザッとメモ。

<!--more-->


GitHub(GHE)  
↓  
webhook 受信  
↓  
Fluentd (td-agent)  
↓  
Elasticsearch  
↓  
Kibana  

（すべて EC2 の同一インスタンスでやる）


### GitHub の webhook 通知を受信する

- webhook を受信して、そのままファイルに出力するエンドポイントを用意する
- [octmmander](https://speakerdeck.com/kenchan/pepabo-ec-tech-meeting-01?slide=8) を使ってサクッとできた
	- octmmander は社内ツールなので非公開だが、[mikedeboer/node-github](https://github.com/mikedeboer/node-github) 等を使えば同様のことが手軽にできる


### Fluentd(td-agent) を使って Elasticsearch に流し込む

in_tail プラグイン

```
<source>
  @type tail
  path /tmp/ghe_events.log
  pos_file /var/log/td-agent/ghe_events.log.pos
  tag issue.event
  format json
  time_key issue.updated_at
</source>
```

Elasticsearch に流し込む

```
<match issue.event>
  @type elasticsearch
  hosts localhost:9200
  logstash_format true
  logstash_prefix ghe
  flush_interval 2s
  type_name issue
</match>
```

Elasticsearch を起動（スペックが低いインスタンスを使ってるのでメモリを指定している）

```
export ES_JAVA_OPTS="-Xms256m -Xmx256m"; bin/elasticsearch
```

### Kibana 

config/kibana.yml のホスト名を変更しないとアクセスできない。


```
server.host: "EC2 のパブリック DNS"
```

![image](https://user-images.githubusercontent.com/1885716/28501793-67d70634-701e-11e7-9112-46d57a606e0d.png)

### その他

webhook を受信したタイミングで、closed イベントの場合に、かかった時間( issue.closed_at - issue.created_at )を計算してログに差し込んでおけば、そのへんの分析もできそう。

