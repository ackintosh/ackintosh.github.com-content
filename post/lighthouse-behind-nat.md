---
title: "Lighthouseが自動的にNAT越えする仕組み"
date: 2022-07-13T17:35:46+09:00
draft: false
---

NATを利用して外部と接続されているプライベートネットワーク内のノードが外部とP2Pの通信を行う場合、NAT越え（NAT Traversal）の対応が必要になる。LighthouseにはUPnPの機能が備わっているが、一応、これを使わなくても自動的にNAT越えをして動作するようになっている。（「一応」と書いたのは、できればUPnPなどの何かしらの設定を行うことが推奨されているので）

<!--more-->

[Advanced Networking - Lighthouse Book](https://lighthouse-book.sigmaprime.io/advanced_networking.html#nat-traversal-port-forwarding)

この、何も設定しなくても自動的にNAT越えする機能がどのようにして実現されているのか調べたので下記に整理していきたい。

まず前提知識としていくつかおさらいしておく。

#### ENR (Ethereum Node Records)

Ethereumのコンセンサスクライアント（旧称: Eth2クライアント）は、ノードの接続情報等を [Ethereum Node Records](https://github.com/ethereum/devp2p/blob/master/enr.md) という形式で表現する。これに [IPアドレス、ポート番号やチェーンの情報等を含めて](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/p2p-interface.md#enr-structure) ノード間で交換している。

ただし上記のENRの仕様に記載されているとおり、IPアドレスやポート番号は「任意」のフィールドである。

#### Node Discovery Protocol v5

Ethereumのコンセンサスクライアントは、[Node Discovery Protocol v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) というプロトコルを利用してノードディスカバリを行っている。Lighthouseでは、このプロトコルのRust実装である [sigp/discv5](https://github.com/sigp/discv5) を利用している。

後に説明するNAT越えは、このプロトコルで規定されている [PONGレスポンス](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#pong-response-0x02) に含まれる `recipient-ip` と `recipient-port` と、 [sigp/discv5](https://github.com/sigp/discv5) に実装されている [IpVote](https://github.com/sigp/discv5/blob/559227b4e62f5113934b8c99cf1355bbb83a6f93/src/service/ip_vote.rs#L10) によって実現されている。

なお、現時点の Node Discovery Protocol v5 の仕様を見る限りでは `IpVote` は規定されていない。なので [sigp/discv5](https://github.com/sigp/discv5) が独自に実装しているのかもしれない。他のコンセンサスクライアントにも同じような実装があるかもしれないが調べていない。

## NAT Traversal

文章でまとめるのが難しいのでシーケンス図に整理した。

<a href="/post/lighthouse-behind-nat.png" target="_blank">![lighthouse-behind-nat.png](/post/lighthouse-behind-nat.png)</a>

