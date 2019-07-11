+++
date = "2019-07-11T09:34:51+09:00"
draft = false
title = "符号なし256bit整数を言語がサポートしていない場合の対応"
+++


Ethereumのスマートコントラクト記述に使われる言語 Solidity では `uint256` をサポートしていてこれがよく使われているが、一方でチェーン側を実装している言語が必ずしもそれに相当する型をネイティブでサポートしているわけではない。

最近 [勉強がてら Plasma Cash の実装](https://github.com/ackintosh/plasma-cash)に手をつけだして、子チェーンを Rust で書き始めたのだけど、まさに Rust は `uint128` までなのでこの壁にぶつかってしまった。とはいえ実際に Rust で実装して動いているプロジェクトはあるので、どんな実装をして `uint256` に対応しているのか、Ethereumに関連するクレートを調べてみた。

<!--more-->

## paritytech/parity-common
parity-common配下の primitive-types クレートに `U256` 構造体が定義されている。

https://github.com/paritytech/parity-common/blob/ed95e273199dd7b63fe113eb76b4dc0f927270df/primitive-types/src/lib.rs#L51-L54

```rust
 construct_uint! {
 	/// 256-bit unsigned integer.
 	pub struct U256(4);
 }
```

なにやら `construct_uint` というマクロで構造体の定義を組み立てていそうなので、マクロの中身を見てみる。

[parity-common/uint/src/uint.rs#L338](https://github.com/paritytech/parity-common/blob/8ebc5c85c2ba606e40525853829c5047499cc5fb/uint/src/uint.rs#L338)

```rust
#[macro_export]
macro_rules! construct_uint {
```

最初に見たの構造体の宣言は[下記](https://github.com/paritytech/parity-common/blob/8ebc5c85c2ba606e40525853829c5047499cc5fb/uint/src/uint.rs#L343)にマッチして展開されていると思われる。  

```rust
( $(#[$attr:meta])* $visibility:vis struct $name:ident ( $n_words:tt ); ) => {
```

構造体の定義に該当するマクロの記述は[ココ](https://github.com/paritytech/parity-common/blob/8ebc5c85c2ba606e40525853829c5047499cc5fb/uint/src/uint.rs#L425)。  

```rust
$visibility struct $name (pub [u64; $n_words]);
```

なので、最初に見た `pub struct U256(4)` は `pub struct U256(pub [u64; 4])` というタプル構造体の宣言に展開される。

このタプル構造体の型定義 `[u64; 4]` は、マクロの定義では `n_words` という名前が付けられているので、これを追っていくことでどんな実装になっているのかが見えてくる。  
勘がいい人は型定義からすでに察しているかもだけど、"`uint256`をどうやって実装しているのか" についての結論は **「符号なし64bit整数を4つ使って表現している」** になる。

例えば、[`uint128` から `U256` を作る関連関数の定義](https://github.com/paritytech/parity-common/blob/8ebc5c85c2ba606e40525853829c5047499cc5fb/uint/src/uint.rs#L347-L352)を見てみると、


```rust
fn from(value: u128) -> $name {
	let mut ret = [0; $n_words];
	ret[0] = value as u64;
	ret[1] = (value >> 64) as u64;
	$name(ret)
}
```

引数で受け取った `u128` の数値を、64bit 2つに分けて、予め用意しておいた4つぶんの配列に桁の小さい方から入れている。（リトルエンディアン）

というわけで謎は解けました。スッキリ。

## rust-bitcoin

ちょっと気になったのでBitcoin関係のクレートも見てみた。

https://github.com/rust-bitcoin/rust-bitcoin/blob/783948446c626ce8b61313f93c3f1c980f475624/src/util/uint.rs

```rust
 macro_rules! construct_uint {
     ($name:ident, $n_words:expr) => (
         /// Little-endian large integer type
         #[repr(C)]
         pub struct $name(pub [u64; $n_words]);
```

parity-commonと結構似ていて、符号なし64bit整数を4つ使って表現している。（Rustの練度が低いので細かな違いを読み解くほどの余裕が無かった）

## go-ethereum

言語が違う(Go)けど、一応有名どころをチェックしておこうということでGethの実装を見たところ、32bit で表現しているようだ。（Goも `uint256` をサポートしていない）

https://github.com/ethereum/go-ethereum/blob/72029f0f88f6263c74efc03eed7f09dd2c249d6a/accounts/abi/numbers.go#L42-L44

```go
 func U256(n *big.Int) []byte {
 	return math.PaddedBigBytes(math.U256(n), 32)
 }
```

それと、こちらはビッグエンディアンで扱っているという違いもあった。

```go
 // ReadBits encodes the absolute value of bigint as big-endian bytes. Callers must ensure
 // that buf has enough space. If buf is too short the result will be incomplete.
 func ReadBits(bigint *big.Int, buf []byte) {
 	i := len(buf)
 	for _, d := range bigint.Bits() {
 		for j := 0; j < wordBytes && i > 0; j++ {
 			i--
 			buf[i] = byte(d)
 			d >>= 8
 		}
 	}
 }
```

