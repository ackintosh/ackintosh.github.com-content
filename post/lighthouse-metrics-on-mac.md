+++
date = "2021-02-10T07:58:27+09:00"
draft = false
title = "Macでlighthouse-metricsをモニタリングする"
tags = ["lighthouse", "Ethereum"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

[GitHub - sigp/lighthouse-metrics: A docker-compose with Grafana + Prometheus for monitoring Lighthouse](https://github.com/sigp/lighthouse-metrics)

上記でdocker-composeを使ってサクッとダッシュボードが見れるようになっているが、[Dockerの "host" ネットワークモードはLinuxのみのサポート](https://docs.docker.com/network/host/)なのでMacで試すことができない。

<!--more-->

なのでdocker-composeを使わず、MacにPrometheusやGrafanaをインストールして作業を進めていたら、Granafaのダッシュボード設定を修正しないといけなかったり(※)したので、手順を [こちら](https://github.com/ackintosh/sandbox/tree/master/lighthouse-metrics) にまとめた。  
(※) 元のdocker-composeを試せていないので、修正のPRを出した方が良いのかわかっていない。

![lighthouse-metrics](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/lighthouse-metrics-on-mac/lighthouse-metrics.png)

ただし、ロードアベレージなどのメトリクスは [Linuxのみのサポート](https://github.com/sigp/lighthouse/blob/59b2247ab898b076ef84ee8d993af7e8f1b867ff/common/eth2/src/lighthouse.rs#L111-L146) なので表示されない。
