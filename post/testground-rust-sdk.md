---
title: "Testground Rust Sdk"
date: 2022-04-02T22:34:02+09:00
draft: false
title: "TestgroundのテストプランをRustで実装する"
tags: ["Rust"]
---

P2Pシステムをテストするフレームワークである [Testground](https://github.com/testground/testground)。最近、[RustのSDKがリリースされた](https://github.com/testground/sdk-rust/releases/tag/v0.1.0)ので試しにテストプランを実装してみた。

<!--more-->

実装したテストプランは [こちら](https://github.com/ackintosh/sandbox/tree/master/rust/crate-testground-sdk-rust) のリポジトリにあげている。

## manifestファイルの作成

下記のように `manifest.toml` を作成する。テストプランのビルドや実行は Docker を利用するので `builders` や `runners` の設定で有効にしている。

```toml
name = "sandbox-sdk-rust"

[defaults]
builder = "docker:generic"
runner = "local:docker"

[builders."docker:generic"]
enabled = true

[runners."local:docker"]
enabled = true

[[testcases]]
name = "main"
instances = { min = 2, max = 2, default = 2 }
```

## テストプランの実装

とりあえず動かしてみるだけなのでシンプルに下記のステップにした。

1. 各ノードがランダムな時間 sleep する
1. sleep が終わったこと( ready であること)を[Sync service](https://docs.testground.ai/concepts-and-architecture/sync-service)に通知する
1. 参加ノードすべてが ready になったら、各ノードは次のステップに進む (テストの成功を Sync service に通知する)

```rust
use std::time::Duration;
use rand::Rng;

#[async_std::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Sync serviceと通信するためのクライアント
    let mut sync_client = testground::sync::Client::new().await?;

    // ランダムな時間 sleep する
    let mut rng = rand::thread_rng();
    let uniform = rand::distributions::Uniform::new(1, 10);
    async_std::task::sleep(Duration::from_secs(rng.sample(uniform))).await;

    let state = "ready".to_string();

    // ready の状態になったことを通知する
    sync_client.signal(state.clone()).await?;

    // 参加ノードすべてが ready の状態になるまで待つ
    sync_client.wait_for_barrier(state, 2).await?;

    sync_client.publish_success().await?;

    Ok(())
}
```

テストプランの実装では Sync service とのやりとりがキモになってくるが、現時点の SDK には、ほぼ上記で使っている `signal` と `wait_for_barrier` くらいしか無い。[この PR](https://github.com/testground/sdk-rust/pull/6) がマージされればだいぶ充実してくるので楽しみ。

個人的にP2Pネットワーキングのテストやシミュレーションはとても興味があるので自分も積極的にコントリビュートしていきたい。

## Dockerfileの作成

"manifestファイルの作成" で述べたとおり、テストプランのビルドや実行に Docker を利用するので、Dockerfile を用意しておく。

```Dockerfile
FROM rust:1.57-bullseye as builder
WORKDIR /usr/src/sdk-rust
COPY . .
RUN cd plan && cargo build

FROM debian:bullseye-slim
COPY --from=builder /usr/src/sdk-rust/plan/target/debug/crate-testground-sdk-rust /usr/local/bin/sandbox
EXPOSE 6060
ENTRYPOINT [ "sandbox"]
```

## テストの実行

インスタンス数 2 で実行してみる。

```shell
$ testground run single \
           --plan=crate-testground-sdk-rust \
           --testcase=main \
           --builder=docker:generic \
           --runner=local:docker \
           --instances=2 \
           --wait

...
...
...
Apr  2 13:32:51.223392  INFO    starting container      {"runner": "local:docker", "run_id": "c9450le49b3ouvtgfdag", "id": "d290b25388bd99d833234e8c31aec531f20fc67de04f479380a31a19698bfbf1", "group": "single", "group_index": 0}
Apr  2 13:32:51.223400  INFO    starting container      {"runner": "local:docker", "run_id": "c9450le49b3ouvtgfdag", "id": "54fa68ae0dd3b6199c1d0c8561b03d5f2a19029d925e06237aaa2d9bc35cc9b8", "group": "single", "group_index": 1}
...
...
...
Apr  2 13:33:00.626327  INFO    9.3978s         OK << single[000] (d290b2) >>
Apr  2 13:33:00.660355  INFO    9.4355s         OK << single[001] (54fa68) >>
...
...
...
Apr  2 13:33:00.940846  INFO    run finished successfully       {"run_id": "c9450le49b3ouvtgfdag", "plan": "sandbox-sdk-rust", "case": "main", "runner": "local:docker", "instances": 2}
```

コンソール出力の `... OK ...` が、ソースコードの `sync_client.publish_success()` の部分になる。

期待通り、Barrier を利用することによって、各ノードが `publish_success()` を行うタイミングを待ち合わせる挙動をしていたのを確認できた。


