+++
date = "2020-12-23T16:55:14+09:00"
draft = false
title = "infura.ioを使ってBeacon Node(lighthouse)を立てる時に遭遇したエラー"
tags = ["lighthouse"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

Beacon Nodeを稼働させるためには、利用するEthereum(eth1)のエンドポイントを指定する必要があるが、ちょっと動作を検証してみたい程度のときに自前でeth1クライアントを立てておくのはやや骨が折れる。そういった場合には infura.io が便利。

[Ethereum API | IPFS API Gateway | ETH Nodes as a Service | Infura](https://infura.io/)


※ [Launch Pad](https://launchpad.ethereum.org/)の手順では、ネットワークの分散のために自前でクライアントを立てることが推奨されているので、本格的な運用の場合は自前で立てる方が良いのだろう。


以下、eth1バックエンドとしてinfura.ioを使ってBeacon Node(lighthouse)を立てようとした時に遭遇したエラーメッセージと、それについて調べたことをメモしておく。

<!--more-->


執筆時点の最新（lighthouse v1.0.4 ）を前提にしている。  
https://github.com/sigp/lighthouse/releases/tag/v1.0.4

## 遭遇したエラーメッセージ

Beacon Nodeを起動してまもなく下記のエラーメッセージが出てきた。

```
Dec 23 10:46:29.801 ERRO Failed to update eth1 cache             error: Failed to update eth1 cache: "All fallback errored: https://goerli.infura.io/v3/{APIキー} => GetDepositLogsFailed(\"Eth1 node returned error: {\\\"code\\\":-32005,\\\"message\\\":\\\"query returned more than 10000 results\\\"}\")", retry_millis: 7000, service: eth1_rpc
```

## どこでエラー出ているのか

エラーログの内容から、場所はすぐに特定できた。

https://github.com/sigp/lighthouse/blob/7933596c891db74e344292e650b05f49673ab830/beacon_node/eth1/src/service.rs#L847

```rust
             /*
              * Step 1. Download logs.
              */
             let block_range_ref = &block_range;
             let logs = endpoints
                 .first_success(|e| async move {
                     get_deposit_logs_in_range(
                         e,
                         &deposit_contract_address_ref,
                         block_range_ref.clone(),
                         Duration::from_millis(GET_DEPOSIT_LOG_TIMEOUT_MILLIS),
                     )
                     .await
                     .map_err(SingleEndpointError::GetDepositLogsFailed)
                 })
                 .await
                 .map_err(Error::FallbackError)?;
```


周辺のコードを読んでいると、下記をしている処理だとわかった。

- eth1のエンドポイントから、[デポジットコントラクト](https://ethereum.org/en/eth2/deposit-contract/)のログを（ブロックのレンジを指定して）[取得している](https://github.com/sigp/lighthouse/blob/d8cda2d86eb3f69185a16d0474b987fbf0b8eb6b/beacon_node/eth1/src/http.rs#L310)
- レンジの幅は、[blocks_per_log_query](https://github.com/sigp/lighthouse/blob/7933596c891db74e344292e650b05f49673ab830/beacon_node/eth1/src/service.rs#L340-L341) の[設定値で決まる](https://github.com/sigp/lighthouse/blob/7933596c891db74e344292e650b05f49673ab830/beacon_node/eth1/src/service.rs#L809-L818)

## なぜエラー出ているのか

infura.ioのドキュメントに、ひとつのクエリでは最大 10,000 の結果を返すとの制限事項書かれている。おそらく、レンジで指定されているブロックに 10,000 を超える数のログが含まれているためにエラーになっているのだろう。

[Infura Documentation | Infura Documentation](https://infura.io/docs/ethereum/json-rpc/ratelimits)

> #### LIMITATIONS
> ...  
> A max of 10,000 results can be returned by a single query


## どうすればよいのか
前述の設定値 `blocks_per_log_query` は、CLIパラメータで指定できるようになっている。（デフォルトは 1,000 ）  
10にしたらエラーが出なくなり、デポジットログのインポートが成功するようになった。（100ではダメだった）

```shell
 $ lighthouse bn \
        --debug-level debug \
        --network pyrmont \
        --eth1-endpoints https://goerli.infura.io/v3/{APIキー} \
        --eth1-blocks-per-log-query 10
```
