---
title: "CopyOnWriteArrayList でリストを安全に更新する"
date: 2017-11-19T17:41:55+09:00
draft: false
tags: ["java"]
---

[デザインパターン入門 マルチスレッド編](http://www.hyuki.com/dp/dp2.html) に、マルチスレッドプログラムの評価基準として `安全性` `生存性` `再利用性` が挙げられている。安全性とはオブジェクトのフィールドが意図した値を保っていることで、安全性が保たれているクラスをスレッドセーフなクラスという。  
マルチスレッドプログラミングにおいてオブジェクトを安全に更新するには、操作が競合しないように synchronized などを使った排他制御の工夫が必要。

<!--more-->

[synchronized メソッドの挙動を JVM のスレッドダンプを見ながら確かめる · 暁](/blog/2017/11/04/java-synchronized/)

以下、 List の操作を例にとって、"スレッドセーフではない" プログラムと CopyOnWriteArrayList クラスを使った改善方法、それから CopyOnWriteArrayList クラスがどのように競合を回避しているかを見ていく。

## スレッドセーフではないプログラム

登場人物は、スレッド間で共有する ArrayList、リストにひたすら書き込む Writer スレッド、リストをひたすら読み込む Reader スレッドの3つ。

```java
import java.util.ArrayList;
import java.util.List;

public class NonThreadSafe {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<Integer>();
        new Writer(list).start();
        new Reader(list).start();
    }
}

class Writer extends Thread {
    private final List<Integer> list;

    public Writer(List<Integer> list) {
        super("Writer");
        this.list = list;
    }

    public void run() {
        for (int i = 0; true; i++) {
            list.add(i);
            list.remove(0);
        }
    }
}

class Reader extends Thread {
    private final List<Integer> list;

    public Reader(List<Integer> list) {
        super("Reader");
        this.list = list;
    }

    public void run() {
        while (true) {
            for (int i : list) {
                System.out.println(i);
            }
        }
    }
}
```

ArrayList クラスはスレッドセーフではないので、これを実行すると `ConcurrentModificationException` が起きる。これは複数のスレッドから読み書きが行われ安全性が失われたことを示すランタイムの例外。  
Reader がリストを読み取ってる最中に Writer がそれを更新してしまうために発生する。

```
Exception in thread "Reader" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
	at sample.Reader.run(NonThreadSafe.java:40)
```

## CopyOnWriteArrayList を使う

ArrayList の代わりに java.util.concurrent.CopyOnWriteArrayList クラスを使う。Writer, Reader はそのままで良い。

```java
import java.util.concurrent.CopyOnWriteArrayList;

public class ThreadSafe {
    public static void main(String[] args) {
        List<Integer> list = new CopyOnWriteArrayList<Integer>();
        new Writer(list).start();
        new Reader(list).start();
    }
}
```

実行すると Reader が読み取った値が出力される 👌

```
...
...
...
33499164
33499194
33499235
Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

## CopyOnWriteArrayList がどうやって競合を回避しているか

クラス名のとおりなのでだいたい想像が付く方が多いかもしれない。

[コピーオンライト - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%94%E3%83%BC%E3%82%AA%E3%83%B3%E3%83%A9%E3%82%A4%E3%83%88)

#### add メソッドの実装

まずは書き込む側の実装を確認する。

[CopyOnWriteArrayList.java#l431](http://hg.openjdk.java.net/jdk/jdk/file/4fab795915b6/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java#l431)

1. 当該クラスが管理しているリストのコピーをつくる
1. 要素を追加する
1. 新しいリストに [差し替える](http://hg.openjdk.java.net/jdk/jdk/file/4fab795915b6/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java#l117)

上記の一連の処理が synchronized ブロックになっているので、ある時点で実行できるスレッドは1つだけに制限される。

#### iterator メソッドの実装

読み込む側の実装を確認する。Reader クラスの拡張 for ループではリストの iterator メソッドが実行されるのでその実装を確認する。

> for (int i : list) {

[CopyOnWriteArrayList.java#l1016](http://hg.openjdk.java.net/jdk/jdk/file/4fab795915b6/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java#l1016)

単純にイテレータを返すだけ。  
また、iterator メソッドや、それが返す [COWIterator](http://hg.openjdk.java.net/jdk/jdk/file/4fab795915b6/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java#l1070) クラスは一切排他制御をしていない。

#### 図にすると

![image](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/copy-on-write-array-list/copy-on-write-array-list.png)

## おわりに

他にも、安全にリスト操作を行う方法として [Collections.synchronizedList](https://docs.oracle.com/javase/jp/8/docs/api/java/util/Collections.html#synchronizedList-java.util.List-) もある。これは CopyOnWriteArrayList と違ってすべての操作を同期的に行う。

スレッドをブロックするのはパフォーマンスの影響が大きいので、Read 操作が多い処理であれば、Read 時にブロックしない CopyOnWriteArrayList の方が適しているだろう。
