+++
date = "2019-10-28T20:11:22+09:00"
draft = false
title = "Libraのステート管理 - Jellyfish Merkle Tree"
tags = ["Libra"]
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

この記事では、Libraネットワーク内で合意が取れたトランザクションを保存する部分の処理を追っていくことで、Libraでアカウントのステート（アカウントに紐づく残高などの情報）をどんな感じに管理しているのかざっくり把握していきたい。

<!--more-->

（以下、https://github.com/libra/libra/tree/e3d03cfe7540479b1e6e75abe9c9e8faf6f8ca32 時点のソースコードを参照している）

## 全体像とStorageモジュール

<a href="https://developers.libra.org/docs/libra-protocol#validator-node-validator" target="_blank">

![Modules](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/modules.png)

</a>

( [FIGURE 1.2 LOGICAL COMPONENTS OF A VALIDATOR.](https://developers.libra.org/docs/libra-protocol#validator-node-validator) )

バリデータノードは上記6つのモジュールで構成されていて、今回処理を追っていくのは諸々のデータの保持を担うStorageモジュール。


StorageモジュールのREADMEにさらに詳しい説明が書かれている。  
[ > READMEs - Storage · Libra](https://developers.libra.org/docs/crates/storage)

<a href="https://developers.libra.org/docs/crates/storage" target="_blank">

![Storage](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/tree.png)

</a>

（わかりやすく図に書き起こされている。赤枠で囲ったあたりの実装を見ていきたい。）

### ドキュメントを読む

https://developers.libra.org/docs/crates/storage#ledger-state

`Ledger State` のセクションに関係することが書かれていて、Sparse Merkle Treeで表現していることと、256bitのSparse Merkle Treeを愚直にやると無駄が多いのでいくつか最適化を施していることがわかる。

https://developers.libra.org/docs/crates/storage#implementation-details

また `Implementation Details` には、永続化にRocksDBを使っていることと、Storageの機能は LibraDB というモジュールを中心に実装されていることが書かれている。

さらに先程のSparse Merkle Treeについても言及されていて、branch nodeとextension node（Ethereumのそれと同等のものと思われる）を使った最適化によって、EthereumのMerkle Patricia Treeよりも短いproof？を生成できるようになっているとのこと。


## ざっくりとソースコードを追っていく

トランザクションに含まれるステートを永続化するまでの流れをざっくりと見ていく。ドキュメントによるとLibraDBを中心に実装されているとのことだったので、コードを眺めていると早速 `save_transactions_impl` というそれっぽい名前のメソッドが見つかった。

https://github.com/libra/libra/blob/9832f16cfa0231178db96d5b8447af003dd6f1cd/storage/libradb/src/lib.rs#L394

```rust
    fn save_transactions_impl(
        &self,
        txns_to_commit: &[TransactionToCommit],
        first_version: u64,
        mut cs: &mut ChangeSet,
    ) -> Result<HashValue> {
```

メソッドの内容を俯瞰してみると、アカウント、イベント、トランザクションの3種類の更新を行っているようなので、アカウントの処理を追っていく。なお、メソッド名から暗示されている通り複数のトランザクションをまとめて処理する実装になっている。（なのでこれ以降に追っていくアカウント関係の処理も複数のステートをまとめて扱うかたちになっているので各種命名に `batch` の単語がよく出てくる）

https://github.com/libra/libra/blob/df7d714e5f6ee695004c45d90ec74e285c973618/storage/libradb/src/state_store/mod.rs#L51

```rust
    pub fn put_account_state_sets(
        &self,
        account_state_sets: Vec<HashMap<AccountAddress, AccountStateBlob>>,
        first_version: Version,
        cs: &mut ChangeSet,
    ) -> Result<Vec<HashValue>> {
```

[StateStore](https://github.com/libra/libra/blob/df7d714e5f6ee695004c45d90ec74e285c973618/storage/libradb/src/state_store/mod.rs#L29-L31)というモジュールに処理が移った。

* JellyfishMerkleTreeを生成してそれにステートを渡すと、Merkle Treeのルートハッシュと TreeUpdateBatch （ステートを適用した結果の増分）が返ってくる
* そのTreeUpdateBatchを使って、LibraDBから参照渡しされている `ChangeSet` を更新する

※ JellyfishMerkleTreeについては後ほど詳しく見ていく

上記を終えたらStateStoreを抜けて、LibraDBに戻る。その後はアカウント関係の処理は特になく、先程 StateStore が更新した `ChangeSet` を使って永続化している。

https://github.com/libra/libra/blob/9832f16cfa0231178db96d5b8447af003dd6f1cd/storage/libradb/src/lib.rs#L370-L372

ということで永続化までざっくり流れを追えた。JellyfishMerkleTreeがステート管理のキモになっていて、これを見ていくことでドキュメントに書かれていた最適化についてもわかってきそう。

![flow](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/flow.png)


## Jellyfish Merkle Tree

Jellyfish Merkle Treeは初めて聞いたのでググってみたがヒットしない。Libraが独自に実装したツリー構造のようだ。

storage/jellyfish-merkle
https://github.com/libra/libra/blob/e3d03cfe7540479b1e6e75abe9c9e8faf6f8ca32/storage/jellyfish-merkle/src/lib.rs#L19


![jellyfish-aa](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/jellyfish-aa.png)


アスキーアートで描かれているクラゲがなかなかのインパクト。rustdocに書かれている説明から下記がわかる。

* InternalNodeとLeafNodeで構成されている
* InternalNodeは、16の子要素を持つ、EthereumのMerkle Patricia Treeのbranch nodeと同じようなもの
* LeafNodeは、EthereumのMerkle Patricia Treeのleaf nodeと同じようなもの
* クラゲの頭（？）の部分がInternalNode
* 触手の部分がLeafNode

（以下、自分なりに実装を読み解いたことをまとめていく）


### InternalNodeの下にInternalNodeがぶら下がることもある
クラゲの絵をパッと見た印象では、InternalNodeの子要素にはLeafNode（または不在を示すNullNode）が入るかと思っていたが、そうではなく、LeafNodeの場合もあるし、InternalNodeの下にInternalNodeがぶら下がることもあるようだ。たしかに、そうじゃないとアカウントのアドレスをツリー構造のパスで効率的に表現できないので、考えてみれば当たり前ではある。


![nodes](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/nodes.png)


### 最適化 - Ethereumとの違い

愚直にSparse Merkle Treeで256bitアドレスのステートを表現しようとすると、巨大で無駄の多いツリーになってしまうので何かしらの最適化が必要になる。例えばEthereumのModified Merkle Patricia Trie場合は、`branch` `leaf` `extension` の3種類のノードがあって、このうち `extension` ノードがパスの共通部分を (`shared nibble(s)`)に持つことでツリーの短縮を行なっている。

* [Patricia Tree · ethereum/wiki Wiki](https://github.com/ethereum/wiki/wiki/Patricia-Tree#modified-merkle-patricia-trie-specification-also-merkle-patricia-tree)
* [blockchain - ELI5 How does a Merkle-Patricia-trie tree work? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/6415/eli5-how-does-a-merkle-patricia-trie-tree-work)

<a href="https://ethereum.stackexchange.com/questions/6415/eli5-how-does-a-merkle-patricia-trie-tree-work" target="_blank">

![modified-mekle-patricia-trie](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/modified-mekle-patricia-trie.png)

</a>


一方でJellyfish Merkle Treeはどうかというと、先述のとおりノードはinternalとleafの2種類。Internal Nodeが、Ethereumでいうところのbranch nodeとextension nodeの役割を兼ねている。

先程のEthereumの図と同じステートをJellyfish Merkle Treeで表現したのが下記。
Ethereumと比べるとextension nodeが無いぶんシンプルになったが、ツリーの階層が深くなってしまった。ただし、この階層の深さはnon-inclusionを効率的に証明する（ = Sparse Merkle Treeとしての性質を満たす）ために必要なのではないかと想像している。（要勉強...!）


![jellyfish-merkle-tree-diagram](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/jellyfish-merkle-tree/jellyfish-merkle-tree-diagram.png)

※ NP: Nibble Path

メモ: 下記、ツリーの動作確認に使ったコード  
(参考: [storage/jellyfish-merkle/src/jellyfish_merkle_test.rs](https://github.com/libra/libra/blob/df7d714e5f6ee695004c45d90ec74e285c973618/storage/jellyfish-merkle/src/jellyfish_merkle_test.rs))


```rust
#[test]
 fn test() {
     // a711355
     let key1 = [
         0xa << 4 | 0x7,
         0x1 << 4 | 0x1,
         0x3 << 4 | 0x5,
         0x5 << 4,
         0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
     ];
     let hash1 = HashValue::new(key1);
 
     // a77d337
     let key2 = [
         0xa << 4 | 0x7,
         0x7 << 4 | 0xd,
         0x3 << 4 | 0x3,
         0x7 << 4,
         0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
     ];
     let hash2 = HashValue::new(key2);
 
     // a7f9365
     let key3 = [
         0xa << 4 | 0x7,
         0xf << 4 | 0x9,
         0x3 << 4 | 0x6,
         0x5 << 4,
         0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
     ];
     let hash3 = HashValue::new(key3);
 
     // a77d397
     let key4 = [
         0xa << 4 | 0x7,
         0x7 << 4 | 0xd,
         0x3 << 4 | 0x9,
         0x7 << 4,
         0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
     ];
     let hash4 = HashValue::new(key4);
 
     let value = AccountStateBlob::from(vec![1u8]);
     let batch = vec![
         (hash1, value.clone()),
         (hash2, value.clone()),
         (hash3, value.clone()),
         (hash4, value.clone()),
     ];
 
     let db = MockTreeStore::default();
     let tree = JellyfishMerkleTree::new(&db);
     let (_root, _batch) = tree.put_blob_set(batch, 0).unwrap();
 
     println!("done");
 }
```

## おわりに

Libraのステート管理まわりのコードを追うことで、Jellyfish Merkle Treeの実装を見つけることができた。（今のところコレについてドキュメントやホワイトペーパーには書かれていない）

Jellyfish Merkle Treeは後発ということもあってか、EthereumのModified Merkle Patricia Trieでは表現できていないnon-inclusionの効率的な証明を考慮した実装になっている。（と言いつつ、ちゃんとEthereumの実装を見たわけではないので正確な比較ではないかもしれない。追って理解を深めていきたい）

