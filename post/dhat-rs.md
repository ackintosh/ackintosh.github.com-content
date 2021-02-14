+++
date = "2021-02-14T15:01:29+09:00"
draft = false
title = "dhat-rsでRustプログラムのヒープメモリ使用量を確認する"
slug = ""
tags = ["Rust"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

[dhat - Rust](https://docs.rs/dhat/)
 
dhat-rsは [DHAT](https://www.valgrind.org/docs/manual/dh-manual.html) と同等の機能を提供するクレートで、 このクレートの [dhat::DhatAlloc](https://docs.rs/dhat/0.2.2/dhat/struct.DhatAlloc.html) をアロケータに設定することで、ヒープの割当を追跡・計測してくれる。

<!--more-->

## 計測する

サンプルとしてLeetCodeの問題を使う。

[Check If Two String Arrays are Equivalent - LeetCode](https://leetcode.com/problems/check-if-two-string-arrays-are-equivalent/)

2つの配列（文字列を要素に持つ配列）が引数で渡され、それが一致するかどうかを返すという問題。素直に実装するなら下記のように1行で済んでしまう。 

```rust
impl Solution {
    pub fn array_strings_are_equal(word1: Vec<String>, word2: Vec<String>) -> bool {
        word1.concat() == word2.concat()
    }
}
```

このプログラムに、下記のように100バイトの文字列（をいくつかに分割した配列）を与えるテストコードで、ヒープメモリの使用量を計測する。

```rust
#[test]
fn heap() {
    let _dhat = Dhat::start_heap_profiling();

    assert!(Solution::array_strings_are_equal(
        vec!["12".into(), "34567890".into(), "123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890".into()],
        vec!["12".into(), "345678901234567890123456789012345678901234567890".into(), "12345678901234567890123456789012345678901234567890".into()],
    ));
}
```

計測結果は 544 bytes だった。

```bash
dhat: Total:     544 bytes in 10 blocks
dhat: At t-gmax: 544 bytes in 10 blocks
dhat: At t-end:  0 bytes in 0 blocks
``` 
 
## ヒープの使用量を改善する
 
先程の解答ではスライスの [concatメソッド](https://doc.rust-lang.org/std/primitive.slice.html#method.concat) を使っているのでコードが簡潔になるが、結合後の文字列が新しく作られるので、それだけ余分にヒープメモリを使っていることになる。

実験として、concatメソッドを使わずに、元の引数の参照を使って配列を比較する解答を実装してみる。

```rust
impl Solution {
    pub fn array_strings_are_equal(word1: Vec<String>, word2: Vec<String>) -> bool {
        let mut w2_index = 0;
        let mut ww2_index = 0;

        for w1 in word1.iter() {
            for i in 0..w1.len() {
                let ww1 = &w1[i..=i];

                if word2.len() <= w2_index {
                    return false;
                }

                let w2 = word2.index(w2_index);
                let ww2 = &w2[ww2_index..=ww2_index];
                if ww1 != ww2 {
                    return false;
                }

                if w2.len() <= (ww2_index + 1) {
                    w2_index += 1;
                    ww2_index = 0;
                } else {
                    ww2_index += 1;
                }
            }
        }

        if word2.len() >= (w2_index + 1) {
            let w2 = &word2[w2_index];
            if w2.len() >= (ww2_index + 1) {
                return false;
            }
        }

        true
    }
}
```

かなりコードの見通しが悪いけど、このプログラムのヒープメモリの使用量は 344 bytes だった。最初のconcatを使った解答と比べると、ちょうど200bytes（100バイトの文字列の引数が2つ分）が減っていることがわかる。

```bash
dhat: Total:     344 bytes in 8 blocks
dhat: At t-gmax: 344 bytes in 8 blocks
dhat: At t-end:  0 bytes in 0 blocks
```

