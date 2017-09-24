+++
date = "2017-09-20T01:51:58+09:00"
draft = false
title = "Java/Kafka 初心者が Kafka Connector を実装するための環境づくり"

+++

Kafka Connect は ver0.9 で実装された、Kafka の入出力を行うためのプラグイン機構のようなもので、 `Source Connector` と `Sink Connector` がある。[多くの Connector が実装されていて](https://www.confluent.io/product/connectors/)、もちろん独自の Connector を実装して利用することもできる。

当記事では、Java/Kafka 初心者(いまの私です)が Connector を実装するための準備として行った環境づくりについて紹介します。

<!--more-->

## archtype を使って土台を生成する

#### archtype とは

Java用プロジェクト管理ツールである Maven で利用する、プロジェクトの雛形。Kafka Connector 用の archtype として下記を使った。

[jcustenborder/kafka-connect-archtype: Maven quick start for building Kafka Connect connectors.](https://github.com/jcustenborder/kafka-connect-archtype)

#### 土台を生成する

IntelliJ IDEA で新規プロジェクトを作るときに `Maven -> Create from archtype -> Add Archetype` で指定する。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/30589698-86aa1ca6-9d76-11e7-9da9-1278aab1782b.png)

`MySourceConnector` `MySinkConnector` という名前で生成される。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/source_tree.png)


## 生成された土台を動かしてみる

コードを書いていく前にまずは生成されたコードをそのまま動かしてみる。

#### Landoop/fast-data-dev

生成されたコードに [docker-compose.yml](https://github.com/jcustenborder/kafka-connect-archtype/blob/master/src/main/resources/archetype-resources/docker-compose.yml) が含まれているが、今回これは使わず [Landoop/fast-data-dev](https://github.com/Landoop/fast-data-dev) を使う。

[Landoop/fast-data-dev](https://github.com/Landoop/fast-data-dev)  
Kafka Docker for development. Kafka, Zookeeper, Schema Registry, Kafka-Connect, Landoop Tools, 20+ connectors

この docker イメージには Kafka とその周辺コンポーネントや、予め ElasticSearch 等いくつかの Connector が入っていて、これらをコマンドひとつで起動できる。また kafka-topics-ui 等の UI ツールが付属していて、開発中にとても重宝する。

#### コンテナを起動する

```
$ docker run --rm -it \
  -p 2181:2181 \
  -p 3030:3030 \
  -p 8081-8083:8081-8083 \
  -p 9581-9585:9581-9585 \
  -p 9092:9092 \
  -e ADV_HOST=127.0.0.1 \
  landoop/fast-data-dev:latest
```

`http://127.0.0.1:3030/` でダッシュボードにアクセスできる。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/dashboard.png)

#### Connector をビルドする

```
$ mvn clean package
```

※ `pom.xml` の `<dependencies>` 要素に依存ライブラリが定義されているので、予めそれらがダウンロードされている必要がある。 IntelliJ IDEA の自動インポートを有効にしておくと良い。  
プロジェクトを開いた時に自動インポートを有効にするかどうか聞かれると思うが、もしスルーしてしまっても `Preferences -> Build Tools -> Maven -> Importing` で有効にできる。

#### ビルドした Connector をマウントした状態でコンテナを起動しなおす

`-v` オプションでビルドされた jar ファイルが出力されているディレクトリを指定する。(ディレクトリ名は適宜読み替える)

```
$ docker run --rm -it \
  -p 2181:2181 \
  -p 3030:3030 \
  -p 8081-8083:8081-8083 \
  -p 9581-9585:9581-9585 \
  -p 9092:9092 \
  -e ADV_HOST=127.0.0.1 \
  -v $(pwd)/target/kafka-connect-colormeshop-1.0-SNAPSHOT-package/share/java:/connectors \
  landoop/fast-data-dev:latest
```

`http://127.0.0.1:3030/logs/connect-distributed.log` で Connector 関連のログが見れる。 ビルドした Connector がロードされている様子がわかるはず。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/log.png)

#### kafka-connector-ui で設定を追加する

Source Connector を追加する。

`http://127.0.0.1:3030/kafka-connect-ui/#/cluster/fast-data-dev/select-connector`

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/connectors.png)

[設定項目 my.setting](https://github.com/jcustenborder/kafka-connect-archtype/blob/master/src/main/resources/archetype-resources/src/main/java/MySourceConnectorConfig.java#L26) が必須なので適当な文字列を入力して `CREATE` 。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/configuration.png)

何事もなく設定を追加できた。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/created.png)

`http://127.0.0.1:3030/logs/connect-distributed.log` で再度 Connector のログを確認するとエラーが出ている。

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/kafka-connector-dev/log2.png)

これは、雛形から生成した状態では [例外を投げるようになっている](https://github.com/jcustenborder/kafka-connect-archtype/blob/master/src/main/resources/archetype-resources/src/main/java/MySourceConnector.java#L39) のが原因なので正しい挙動。

## 環境ができた

これで試行錯誤するための環境ができたのであとは実装していくのみ！  

