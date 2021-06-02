+++
title = "sigp/discv5 v0.1.0-beta.4で追加された機能"
date = 2021-06-02T07:11:28+09:00
draft = false
slug = ""
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

[Release v0.1.0-beta.4 · sigp/discv5 · GitHub](https://github.com/sigp/discv5/releases/tag/v0.1.0-beta.4)

個人的に気になった機能のソースコードを追ったのでメモ。

<!--more-->


## 同一サブネットのノード数を制限する

> Adds an abstract filter to the local routing table, allowing easier ways to filter entries from being added to the table. This is currently used for creating limits on IP ranges and incoming/vs outgoing nodes.

以前から、怪しいノードを [BANする機能](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/socket/filter/mod.rs#L154-L155) は実装されていたが、それとは別にIPアドレスベースでフィルタする機能が実装された。


#### 機能

- ルーティングテーブル内に同じ /24 サブネットに属するノード数の上限を設ける
- テーブル単位、バケット単位で上限が設定される

#### 実装

同じ /24 サブネットに属するノード数が制限値内であるかどうかを確認している

- https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/kbucket/filter.rs#L70-L96

2種類のフィルタ

- [IpTableFilter](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/kbucket/filter.rs#L41)
  - KBucketsTableで利用するフィルタ
  - 「同じ /24 サブネットに属するノード数は 10 まで」に[設定されている](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/kbucket/filter.rs#L35-L36)
  - フィルタ自体には、（設定値以外は）ルーティングテーブルに関係する実装は入っていない。KBucketsTableのフィールドに保持されて[使われる](https://github.com/sigp/discv5/blob/b1e7ad688228c85e802abc4b165783e4e95db886/src/kbucket.rs#L344)
- [IpBucketFilter](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/kbucket/filter.rs#L54)
  - Bucketで利用するフィルタ
  - 「Bucket内に、同じ /24 サブネットに属するノード数は 2 まで」に[設定されている](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/kbucket/filter.rs#L37-L38)

#### 設定値のカスタマイズはできない

- ip_limitが設定されている場合にこの制限が有効になる
- https://github.com/sigp/discv5/blob/b1e7ad688228c85e802abc4b165783e4e95db886/src/discv5.rs#L95-L105
- 上記のコメントにかかれているとおり、現状、このフィルタのカスタマイズはできない


## セッションに "接続方向" を持たせる

> Introduces the concept of connection direction. Sessions that get established are now tagged with either being ingoing or outgoing. This can be used to limit the number of incoming connections allowed per bucket.

- 各ノードとの間のセッションに InComming/Outgoing の方向を持たせて、フィルタリングに利用する

#### 実装

- [ConnectionDirectionの定義](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/handler/mod.rs#L116-L123)
- ConnectionDirectionを保持する構造体
  - [NodeStatus](https://github.com/sigp/discv5/blob/b1e7ad688228c85e802abc4b165783e4e95db886/src/kbucket/bucket.rs#L54-L60)
- [Directionを判断しているコード](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/handler/mod.rs#L676-L682)
- ConnectionDirection::InComming はフィルタリングに使われている
  - [例](https://github.com/sigp/discv5/blob/b1e7ad688228c85e802abc4b165783e4e95db886/src/kbucket/bucket.rs#L308-L311)


## その他の改善

> Improves and adds some extra filtering logic to harden the security and limit the allowed actions of malicious actors.

- 該当すると思われるコミット
  - https://github.com/sigp/discv5/pull/66/commits/c9fe1e15a208410494377bf6e7a1634608fb4220
- 同一IPあたりのノード数の制限を追加
  - 制限を超過したIPアドレスは[BANリストに追加される](https://github.com/sigp/discv5/blob/82c11b261fc65f0962f79612393a8ce2a4ccb01d/src/socket/filter/mod.rs#L195-L217)
  - デフォルトの設定値は 10
- IpVoteのエントリに有効期限が追加された

