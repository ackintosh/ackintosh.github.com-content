+++
date = "2016-04-04T12:44:06+09:00"
draft = false
title = "Snidel 0.5 をリリースしました"
+++

引き続き内部のリファクタリングに加えて、  
今回のリリースでは Snidel::get() の返り値を変更しました。

<a href="https://github.com/ackintosh/snidel/releases/tag/0.5.0" target="_blank">Release 0.5.0 · ackintosh/snidel</a>

<!--more-->

これまでは子プロセス群が返す値を単純に配列で返していたのですが  
バージョン 0.5 では配列の代わりに Snidel\ForkCollection のインスタンスを返します。

```
// Snidel::get() returns instance of Snidel\ForkCollection
$forkCollection = $snidel->get();
```

Snidel\ForkCollection は Snidel\Fork の集合で、  
Snidel\Fork が子プロセスの実行結果についての詳細を把握しています。

```
// Snidel\ForkCollection implements \Iterator
foreach ($forkCollection as $fork) {
    echo $fork->getPid();// プロセスID
    echo $fork->getOutput();// 標準出力
    echo $fork->getReturn();// 返り値
}
```

Snidel\ForkCollection::toArray() を使えばこれまで通り、返り値だけを配列で取得することもできます。


```
var_dump($forkCollection->toArray());
// array(3) {
//   [0]=>
//   string(3) "bar"
//   [1]=>
//   string(3) "foo"
//   [2]=>
//   string(3) "baz"
// }
```

また、ソースコードの静的解析ツール Scrutinizer のスコアが  
9.56 まで改善したのでこの場で静かにアピールしておきたいと思います。

![scrutinizer](https://dl.dropboxusercontent.com/u/22083548/octopress/snidel_0_5_0.png)