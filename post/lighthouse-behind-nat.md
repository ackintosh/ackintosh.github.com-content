---
title: "Lighthouseノードが自分の公開アドレスを自動的に認識する仕組み"
date: 2022-07-13T17:35:46+09:00
draft: false
---

例えば一般的なネット環境下にあるラップトップでLighthouseノードを起動した場合、NATの背後でノードが稼働することになる。この場合、ノードは自力で自分の公開IPアドレス/ポートを知ることは難しい。また、インターネットプロバイダの都合により稼働中に公開アドレスが変更になる可能もある。

Lighthouseには公開IPアドレス/ポートを自動的に認識する仕組みがあるので、どのようにして実現されているのか調べた。

<!--more-->

まず前提知識としていくつかおさらいしておく。

#### ENR (Ethereum Node Records)

Ethereumのコンセンサスクライアント（旧称: Eth2クライアント）は、ノードの接続情報等を [Ethereum Node Records](https://github.com/ethereum/devp2p/blob/master/enr.md) という形式で表現する。これに [IPアドレス、ポート番号やチェーンの情報等を含めて](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/p2p-interface.md#enr-structure) ノード間で交換している。

ただし上記のENRの仕様に記載されているとおり、IPアドレスやポート番号は「任意」のフィールドである。

#### Node Discovery Protocol v5

Ethereumのコンセンサスクライアントは、[Node Discovery Protocol v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) というプロトコルを利用してノードディスカバリを行っている。Lighthouseでは、このプロトコルのRust実装である [sigp/discv5](https://github.com/sigp/discv5) を利用している。

後に説明する仕組みは、このプロトコルで規定されている [PONGレスポンス](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md#pong-response-0x02) に含まれる `recipient-ip` と `recipient-port` と、 [sigp/discv5](https://github.com/sigp/discv5) に実装されている [IpVote](https://github.com/sigp/discv5/blob/559227b4e62f5113934b8c99cf1355bbb83a6f93/src/service/ip_vote.rs#L10) によって実現されている。

なお、現時点の Node Discovery Protocol v5 の仕様を見る限りでは `IpVote` は規定されていない。なので [sigp/discv5](https://github.com/sigp/discv5) が独自に実装しているのかもしれない。他のコンセンサスクライアントにも同じような実装があるかもしれないが調べていない。

## 公開アドレスを認識する仕組み

文章でまとめるのが難しかったためシーケンス図に整理したので参照されたい。

補足：下図のノードAとノードBが直接通信できるかどうかは、NAT超えの課題があるので、まだ別の問題になる。

<a href="/post/lighthouse-behind-nat.png" target="_blank">![lighthouse-behind-nat.png](/post/lighthouse-behind-nat.png)</a>

## 2022-07-28 追記

関連するトピックとして、NAT越えのための [Rendezvous protocol というのが議論されて](https://github.com/ethereum/devp2p/issues/207) いて、まだ仕様は固まっていないが先んじて sigp/discv5 で[実装が進められている](https://github.com/sigp/discv5/pull/134)。

