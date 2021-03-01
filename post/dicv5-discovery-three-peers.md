+++
date = "2021-02-28T23:38:19+09:00"
draft = false
title = "sigp/discv5のテストコードを読んでノードの動作を理解する"
tags = ["discv5"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

[sigp/discv5](https://github.com/sigp/discv5/)のとあるテストを読み解きながら、各ノードがどんな通信をしているのか、各ノードのk-bucketの中身はどうなっているのかを確認していくことで理解を深める。

<!--more-->

なお、sigp/discv5のソースコードは執筆時点で最新の [02d2c896c6](https://github.com/sigp/discv5/tree/02d2c896c66f8dc2b848c3996fedcd98e1dfec69) 、 [discv5の仕様](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) は v5.1 を参照する。

## 対象のテストコード

今回読むテストコードは [こちら](https://github.com/sigp/discv5/blob/f47dac2e1f59735bb0affe3502372c21d758cc4e/src/discv5/test.rs#L162) で、FINDNODEリクエストに対して期待通りのノード数が得られるかを確認するテスト。

## ノードの起動

https://github.com/sigp/discv5/blob/f47dac2e1f59735bb0affe3502372c21d758cc4e/src/discv5/test.rs#L169

- bootstrap node
- 通常のnodeを3つ (node1, node2, node3)
- target node (FINDNODEリクエストのターゲットとして指定するノード)

の合計5つのノードを起動する。

ここでのポイントは、ノードを起動する際に予め、bootstrap nodeと、その他のノード（node1, node2, node3, target node）の距離が "256" になるようにseed値を固定している。これによって、FINDNODEリクエストの結果にバラつきが出ないようにしている。

discv5における [ノード間の距離](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#nodes-records-and-distances) とは、ノードID(32byte)のXORの結果を対数で表したもので、sigp/discv5の実装では [leading zeroの数を利用している](https://github.com/sigp/discv5/blob/f75da8aca06a814f987711188e7eb250b9c2fe6f/src/kbucket/key.rs#L92)。

各ノードのノードID(途中省略)は下記。

- bootstrap node: 0x2390..81d9
- node1: 0xb862..63b1
- node2: 0xbd03..5d70
- node3: 0xe030..dcbe
- target node: 0xf47b..3704

## node1〜3にbootstrapノードを登録する

https://github.com/sigp/discv5/blob/f47dac2e1f59735bb0affe3502372c21d758cc4e/src/discv5/test.rs#L186-L199

- bootstrap nodeに、node1, 2, 3のENRを登録する
- node1, 2, 3に、bootstrap nodeのENRを登録する

前述の通り、bootstrapノードと、その他のノード（node1〜3、target node）の距離が "256" になるように調整されているので、各ノードのk-bucketの 256 番目のbucketに登録される。

この時点での、起動しているノードとそれらの繋がり（add_enrによって認識しているノード）は下記のようになっている

![initial_kbuckets](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/dicv5-discovery-three-peers/initial_kbuckets.png)

### node1 から FINDNODEリクエストを送信する

https://github.com/sigp/discv5/blob/f47dac2e1f59735bb0affe3502372c21d758cc4e/src/discv5/test.rs#L204-L209

メッセージの仕様

- [FINDNODE](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#findnode-request-0x03)
- [NODES](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#nodes-response-0x04)

シーケンス図

![sequence](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/dicv5-discovery-three-peers/sequence.png)
