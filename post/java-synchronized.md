---
title: "synchronized メソッドの挙動を JVM のスレッドダンプを見ながら確かめる"
date: 2017-11-04T17:56:28+09:00
draft: false
tags: ["java"]
---

最近、趣味で Java 製プロダクトをいじっていたり、[デザインパターン入門マルチスレッド編](https://www.amazon.co.jp/%E5%A2%97%E8%A3%9C%E6%94%B9%E8%A8%82%E7%89%88-Java%E8%A8%80%E8%AA%9E%E3%81%A7%E5%AD%A6%E3%81%B6%E3%83%87%E3%82%B6%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3%E5%85%A5%E9%96%80-%E3%83%9E%E3%83%AB%E3%83%81%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E7%B7%A8-%E7%B5%90%E5%9F%8E-%E6%B5%A9/dp/4797331623)を読んでいることもあって Java のコードを書くようになった。  
これまでほぼ PHP しかやってこなかったので [java.util.concurrent パッケージ](https://docs.oracle.com/javase/jp/7/api/java/util/concurrent/package-summary.html) の充実っぷりに衝撃をうけた。これらのクラスを使って分散アルゴリズムの実装に挑戦してみたい。

<!--more-->

今回はスレッドの排他制御の仕組みである synchronized メソッドの挙動(ブロックされる範囲)を、 JVM のスレッドダンプを見ながら確かめてみる。挙動を確かめるだけならコードを動かすだけで充分なのだが、今後も Java を書くならスレッドダンプを見ることに慣れておいたほうが良いだろうということで。

### コード

よくある銀行口座から引き落とすコード。引き落としを行う `withdraw()` は synchronized メソッドになっているので、ある時点で実行できるスレッドは1つだけ。  
引き落とし処理には約10秒かかるので、その間もう片方のスレッドがブロックされているか否かがスレッドダンプから見て取れるはず。

```java
class Bank {
    private int money;

    public Bank(int money) {
        this.money = money;
    }

    public synchronized boolean withdraw(int m) {
        System.out.println("引き落とし中...");

        try {
            Thread.sleep(10 * 1000);
        } catch (InterruptedException e) {
            System.out.println(e);
        }

        if (money < m) {
            return false;
        }

        money -= m;
        System.out.println("引き落とし完了!");
        return true;
    }
}

class Withdraw extends Thread {
    private Bank bank;

    public Withdraw(Bank bank) {
        this.bank = bank;
    }

    @Override
    public void run() {
        if (!bank.withdraw(1000)) {
            System.out.println("口座残高が足りません!");
        }
    }
}
```



### 同一インスタンスの synchronozed メソッド呼び出しはブロックされる

###### コード

```java
public class Sample {
    public static void main(String[] args) {
        Bank bank = new Bank(1000);
        new Withdraw(bank).start();
        new Withdraw(bank).start();
    }
}
```

###### スレッドダンプ

スレッドダンプに、スレッドの状態を表す [Thread.State](https://docs.oracle.com/javase/jp/8/docs/api/java/lang/Thread.State.html) が出力されるので、これを確認する。  
Thread-1 が `BLOCKED` なのでブロックされている。

```
$ jps | grep Sample | cut -d ' ' -f 1 | xargs -I{} jstack {}
...
...
...


"Thread-1" #12 prio=5 os_prio=31 tid=0x00007ff9cb87b000 nid=0x5b03 waiting for monitor entry [0x000070000b3b2000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at Bank.withdraw(Sample.java:33)
        - waiting to lock <0x000000076ac29250> (a Bank)
        at Withdraw.run(Sample.java:19)

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007ff9cb87f000 nid=0x5903 waiting on condition [0x000070000b2af000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at Bank.withdraw(Sample.java:36)
        - locked <0x000000076ac29250> (a Bank)
        at Withdraw.run(Sample.java:19)

...
...
...
```

### 異なるインスタンスの synchronozed メソッド呼び出しはブロックされない

###### コード

```java
public class Sample {
    public static void main(String[] args) {
        Bank bank = new Bank(1000);
        Bank bank2 = new Bank(1000);
        new Withdraw(bank).start();
        new Withdraw(bank2).start();
    }
}
```


###### スレッドダンプ

Thread-1, Thread-0 どちらも `TIMED_WAITING` で、今回でいうところの引き落とし処理中であることがわかる。

```
$ jps | grep Sample | cut -d ' ' -f 1 | xargs -I{} jstack {}
...
...
...

"Thread-1" #12 prio=5 os_prio=31 tid=0x00007f9e84819000 nid=0x5b03 waiting on condition [0x0000700002cf9000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at Bank.withdraw(Sample.java:36)
        - locked <0x000000076ac29100> (a Bank)
        at Withdraw.run(Sample.java:19)

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007f9e8300d800 nid=0x5903 waiting on condition [0x0000700002bf6000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at Bank.withdraw(Sample.java:36)
        - locked <0x000000076ac290f0> (a Bank)
        at Withdraw.run(Sample.java:19)

...
...
...
```

### クラスメソッドの場合

`withdraw()` をクラスメソッドにした場合。

###### コード

```java
public class Sample {
    public static void main(String[] args) {
        new Withdraw().start();
        new Withdraw().start();
    }
}

class Bank {
    private static int money = 1000;

    public static synchronized boolean withdraw(int m) {
        System.out.println("引き落とし中...");

        try {
            Thread.sleep(10 * 1000);
        } catch (InterruptedException e) {
            System.out.println(e);
        }

        if (money < m) {
            return false;
        }

        money -= m;
        System.out.println("引き落とし完了!");
        return true;
    }
}

class Withdraw extends Thread {
    @Override
    public void run() {
        if (!Bank.withdraw(1000)) {
            System.out.println("口座残高が足りません!");
        }
    }
}
```

###### スレッドダンプ

もちろんブロックされている。

```
"Thread-1" #12 prio=5 os_prio=31 tid=0x00007fe10b86a000 nid=0x5b03 waiting for monitor entry [0x000070000412c000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at Bank.withdraw(Sample.java:36)
        - waiting to lock <0x000000076aed9680> (a java.lang.Class for Bank)
        at Withdraw.run(Sample.java:21)

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007fe10b869800 nid=0x5903 waiting on condition [0x0000700004029000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at Bank.withdraw(Sample.java:39)
        - locked <0x000000076aed9680> (a java.lang.Class for Bank)
        at Withdraw.run(Sample.java:21)
```

### 異なるインスタンスでクラスメソッドを実行した場合

※ インスタンスでクラスメソッドを実行するのはそもそも良くないコードだが、挙動の確認として。

###### コード

```java
public class Sample {
    public static void main(String[] args) {
        Bank bank = new Bank();
        Bank bank2 = new Bank();
        new Withdraw(bank).start();
        new Withdraw(bank2).start();
    }
}

/** Bank クラスは同じなので省略 **/

class Withdraw extends Thread {
    private Bank bank;

    public Withdraw(Bank bank) {
        this.bank = bank;
    }

    @Override
    public void run() {
        if (!bank.withdraw(1000)) {
            System.out.println("口座残高が足りません!");
        }
    }
}
```

###### スレッドダンプ

「クラスメソッドの場合」と同様にブロックされている。

```
"Thread-1" #12 prio=5 os_prio=31 tid=0x00007fedf1031800 nid=0x5b03 waiting for monitor entry [0x00007000066d9000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at Bank.withdraw(Sample.java:36)
        - waiting to lock <0x000000076ac28d78> (a java.lang.Class for Bank)
        at Withdraw.run(Sample.java:21)

"Thread-0" #11 prio=5 os_prio=31 tid=0x00007fedf008e800 nid=0x5903 waiting on condition [0x00007000065d6000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at Bank.withdraw(Sample.java:39)
        - locked <0x000000076ac28d78> (a java.lang.Class for Bank)
        at Withdraw.run(Sample.java:21)
```