---
title: "Kafka Connect ColormeShop でカラーミーショップの受注をリアルタイムに分析する"
date: 2017-10-07T20:02:45+09:00
draft: false
---

最近 Apache Kafka をいじっていて、だいたい概要がわかってきたのでそろそろ自分でコネクタを実装してみようということで作った。 

[ackintosh/kafka-connect-colormeshop](https://github.com/ackintosh/kafka-connect-colormeshop)

> Kafka Connect connector for reading data in real time from ColormeShop

<!--more-->

先日、カスタムコネクタの実装を始めるための環境準備について [Java/Kafka 初心者が Kafka Connector を実装するための環境づくり](/blog/2017/09/20/kafka-connector-dev/) にまとめていた。  
当記事では、kafka-connect-colormeshop を使ってカラーミーショップの受注をリアルタイムに分析する方法を紹介する。


## コネクタの実装について

Kafka Connect には `Source Connector` と `Sink Connector` があって、今回実装したのは `Source Connector` の方。つまり、[カラーミーショップAPI](https://shop-pro.jp/func/api/)からデータを取ってきて Kafka トピックのレコードとして保存する役割。

Connector の中身は `Connector` `ConnectorConfig` `Task` の3つで構成されている。

- `Connector` : Kafka Connect のワーカープロセスの開始/終了時の挙動を定義する。
- `ConnectorConfig` : コネクタが扱う設定項目について型・必須/任意・説明を定義する。
- `Task` : おそらくコネクタの実装で一番作り込むところ。ワーカーが行う具体的な処理を定義する。


[環境づくり](/blog/2017/09/20/kafka-connector-dev/)で書いたとおりに雛形から土台を生成したあと、上記3パーツの `TODO:` になっている実装を埋めていくかたちになる。今回作った kafka-connect-colormeshop で言うと下記。

- `Connector` : (特にやることなかった)
- `ConnectorConfig` : ワーカーの動作に必要な API アクセストークンや、取得した受注データの保存先となる Kafka トピック名といった設定項目を定義する。  
- `Task` : とってきた受注データを、予め定義したスキーマに合わせて組み立てる。

上記を見ていただいてわかるとおり、やってることはとてもシンプル。

## 使い方

`docker compose up` すると、Kafka クラスタと各種 UI ツールが起動する。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/ui-top.png)

KAFKA CONNECT UI で `ColormeShopSourceConnector` を選択し、保存先のトピック名と API のアクセストークンを入力する。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/kafka-connect-ui.png)

これで kafka-connect-colormeshop の設定は終わり。UI のトップに戻って今度は KAFKA TOPICS UI を開くと、新規受注がトピックに流れ込んできているのがわかる。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/kafka-topics-ui.png)

## カラーミーショップの受注をリアルタイムに分析する

Kafka Connect には、Kafka の開発を主導している confluent 社をはじめとした各社やコミュニティによって様々なコネクタが開発されている。

[Kafka Connect | Confluent](https://www.confluent.io/product/connectors/)

以下では、kafka-connect-colormeshop と既存のコネクタを組み合わせてカラーミーショップの受注データを Elasticsearch に保存し、Kibana を使って都道府県別の購入者数を可視化する。

([docker-compose.yml](https://github.com/ackintosh/kafka-connect-colormeshop/blob/master/docker-compose.yml) に Elasticsearch と Kibana も含まれているので別途用意する必要はない)

#### Elasticsearch コネクタの設定

KAFKA CONNECT UI で Elasticsearch コネクタを選択する。2種類あるが confluent 版を使う。  

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/elasticsearch-connector.png)

トピック名や ES サーバーの URL 指定する。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/elasticsearch-connector-configuration.png)


#### Kibana の Region Map で可視化する

Region Map は 5.5.0 で実装された、国や都市の単位で集計・可視化する機能。デフォルトでは、 国単位 `World Countries` とアメリカの州単位 `US States` に対応している。  
オリジナルの単位で集計したい場合は、GeoJSON を用意して [設定ファイルで指定](https://www.elastic.co/guide/en/kibana/current/settings.html#regionmap-settings) する。

- [Region Maps | Kibana User Guide [5.6] | Elastic](https://www.elastic.co/guide/en/kibana/current/regionmap.html)
- [Region Map で都道府県のデータを描画してみた #Kibana ｜ Developers.IO](http://dev.classmethod.jp/server-side/elasticsearch/kibana-region-map-custom-geojson/)

都道府県の描画に必要な設定はリポジトリ内の [kibana.yml](https://github.com/ackintosh/kafka-connect-colormeshop/blob/master/kibana.yml) に予め設定してあるので、下記を行うだけで良い。

Data タブの Field で `customer.pref_id`を選択する。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/regionmap-data.png)

Option タブの Layer Settings で `Japan` `Prefecture id` を選択する。

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/regionmap-option.png)

「▶」を押すと集計結果が地図上に描画される。かっこいい！

![](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-colormeshop-real-time-analytics/regionmap-japan.png)


