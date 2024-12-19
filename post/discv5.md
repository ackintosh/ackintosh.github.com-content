---
title: "Node Discovery Protocol v5 〜Beacon Node間の接続メカニズム〜"
date: 2024-12-17T22:04:31+09:00
draft: false
---

[Ethereum Advent Calendar 2024](https://qiita.com/advent-calendar/2024/ethereum) 17日目の記事です。

Ethereumは世界中に分散する不特定多数のノードによって形成されるP2Pネットワークです。本記事ではEthereumのコンセンサスレイヤを担うBeacon Nodeにおいて、ノード同士がお互いを発見・接続するメカニズムであるNode Discovery Protocol v5について概要を解説します。

<!--more-->

## なぜNode Discoveryが必要なのか

まずはNode Discoveryの必要性を理解するため、クライアント・サーバモデルとP2Pネットワークを比較してみましょう。

### クライアント・サーバモデル

![client-server](https://s3.ap-northeast-1.amazonaws.com/ackintosh.github.io/discv5/client-server.png)

> _クライアント・サーバモデル（ChatGPTで生成）_

クライアント・サーバモデル（典型的なWebサービスなど）ではクライアントとサーバが明確に役割分担されています。クライアントはサーバにリクエストを送り、サーバは処理結果を返す仕組みです。このモデルではクライアントが事前にサーバのアドレスを知っていることが前提で、そのアドレスは多くの場合は https で始まるURLとして表現されています。また、サーバは一般的に特定の企業や団体によって集中管理されています。

### P2Pネットワーク

![p2pnetwork](https://s3.ap-northeast-1.amazonaws.com/ackintosh.github.io/discv5/p2pnetwork.webp)

> _P2P（ChatGPTで生成）_

対照的にP2Pネットワーク（例：Ethereum）は、全てのノードが平等で役割分担がありません。また、Ethereumのようにパブリック型の場合、誰でもノードを立ててネットワークに参加させることができます。このため、不特定多数のノードが参加するコンピュータネットワークで、全体を集中管理するサーバや管理者が存在せず、各ノードが自律分散的に動作してEthereumのネットワークを形成します。

Ethereumは執筆時点で[10,000以上のノードが稼働](https://ccaf.io/cbnsi/ethereum/network_analytics)していて、ひとつのノードは数十〜百程度のノードと通信をしながら動作します。不特定多数が参加するネットワークですので、通信相手のノードが急にオフラインになったり、帯域が細いなどの理由で不安定なノードもいますし、挙動の怪しいノードはBANすることもあります。そのため接続相手のノードは常に一定とは限らず、入れ替わりが頻繁に生じます。この際、新たな通信相手を効率的に見つける手段が必要になります。全体を集中管理する存在がいればそこに問い合わせれば良いのですが不特定多数が参加するパブリック型ではそれが不可能です。そこで必要になるのがNode Discoveryで、Ethereum（のコンセンサスレイヤ）ではNode Discovery Protocol v5というプロトコルが実装されています。

## Node Discovery Protocol v5

Node Discovery Protocol v5（以下、discv5）は分散ハッシュテーブルの手法である [Kademlia](https://ja.wikipedia.org/wiki/Kademlia) を基に、Ethereum向けに設計されたプロトコルです。

### Node Discoveryの流れ

下記のシーケンス図のように、既知のノードから他のノードを紹介してもらう形で探索が進みます。

![node-discovery](https://s3.ap-northeast-1.amazonaws.com/ackintosh.github.io/discv5/node-discovery.png)

> _説明用に簡略化しているため、実際のdiscv5の挙動とは一部異なります_

- ノードAはノードEと通信したいが、連絡先を知らないため、ノードBに紹介を依頼
- ノードBはEを知らないが、代わりにEに最も近いノードCの連絡先を返す
- ノードAは返ってきた情報をもとに探索を繰り返し、最終的にEと通信可能になる

この探索を実現するために重要なのが、 *ノード間の「距離」を定量化する仕組み* です。

### ノード間の距離

「距離」はノードの物理的な位置ではなく、ノード識別子（ノードID）同士の差異を指します。ノードIDは公開鍵から導出される32バイトのデータで、下記の計算式で距離を定義します。

[Nodes, Records and Distances](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-theory.md#nodes-records-and-distances)

> distance(n₁, n₂) = n₁ XOR n₂  
> logdistance(n₁, n₂) = log2(distance(n₁, n₂))

簡単のため3ビットで具体例を示すと下記のようになります。

**例1**  
ノードA = 001  
ノードB = 010  
とすると、  
→ XOR：011  
→ 距離：2  

**例2**  
ノードA = 001  
ノードA  = 001  
とすると、  
→ XOR：000  
→ 距離：0（同一ノード）  

なお、上記で計算している対数距離は log2(N) ではなく、シンプルに先頭のゼロの数を使って計算されています。

> IDのビット長 - XORの先頭ゼロの数  

**例1の場合**   
3ビット - 1 = 2

これはビット単位の距離を効率的に計算するために使われる手法で分散システムではよく使われていると思われます。実際のノードIDは32バイトですのでその広大な空間に散らばるID同士を効率的に計算できるメリットは大きいです！

### 補足：boot nodes

「Node Discoveryの流れ」で示したシーケンス図は、予めノードAが最低でも1つは他ノードを知っていないと成立しません。ですので実際のEthereumノードは起動時に事前に定義されたノード（boot nodes）にアクセスしてNode Discoveryを開始します。この挙動はdiscv5の仕様外で、ノード実装に委ねられています。

Ethereum Foundationや各ノードの開発チームが運用しているboot nodesの情報が[こちら](https://github.com/eth-clients/mainnet/blob/main/metadata/bootstrap_nodes.yaml)で管理・公開されています。基本的にはノード起動時にこれらのboot nodesにアクセスすることでNode Discoveryを開始します。

## おわりに

この記事では、P2Pに馴染みのないかたを想定してNode Discoveryの必要性やその仕組み、ノード間の距離を定量化する手法をご紹介しました。不特定多数のノードが自律分散的に協調する裏でこのような仕組みが動いていると考えると大変興味深いですね！

簡潔さを重視したので特にシーケンス図はNode Discovery Protocol v5の動作を正確に表しているわけではありませんのでご了承ください。もしこの記事を読んで面白そうと思っていただけたら、[仕様](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md)や実装を眺めてみると新しい発見があるかもしれません。いくつかの言語で実装されていますが、私は[Rust実装](https://github.com/sigp/discv5)のメンテナのひとりですので何か不明点があればお答えできるかもしれません。

