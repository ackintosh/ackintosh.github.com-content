+++
date = "2018-11-06T00:25:24+09:00"
draft = true
title = "入れ子クラスとBuilderパターン"
slug = ""
tags = ["Java"]
image = ""
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

これまでPHPメイン・Ruby少々という感じだったのだが、諸々環境の変化でJavaをちゃんと使えるようになろうという機運が高まったので勉強(といったら大げさかもだが)し始めたのでついでにアウトプットする。

<!--more-->

## 入れ子クラス

[Javaの文法 - Wikipedia](https://ja.wikipedia.org/wiki/Java%E3%81%AE%E6%96%87%E6%B3%95)を眺めていたら [ネストされたクラスを作ることができる](https://ja.wikipedia.org/wiki/Java%E3%81%AE%E6%96%87%E6%B3%95#.E3.82.AF.E3.83.A9.E3.82.B9) とあって、少し興味を惹かれたので調べた。

[Nested Classes (The Java™ Tutorials > Learning the Java Language > Classes and Objects)](https://docs.oracle.com/javase/tutorial/java/javaOO/nested.html)

入れ子クラス(Nested Classes)はクラスの中で定義されたクラスの総称で、staticか否かで大きくふたつに分けられる。staticなものを静的入れ子クラス(static nested classes)、非staticは内部クラス(inner classes)と呼ぶ。

#### 静的入れ子クラス(static nested classes)

```java
class OuterClass {
    static class StaticNestedClass {
    }
}

OuterClass.NestedClass nestedObject = new OuterClass.StaticNestedClass();
```

#### 内部クラス(inner classes)

```java
class OuterClass {
    class InnerClass {
    }
}

OuterClass outerObject = new OuterClass();
OuterClass.InnerClass innerObject = outerObject.new InnerClass();
```

さらに内部クラスには、ローカルクラス(local classes)と匿名クラス(anonymous classes)という仲間がいる。

#### ローカルクラス(local classes)

[Local Classes (The Java™ Tutorials > Learning the Java Language > Classes and Objects)](https://docs.oracle.com/javase/tutorial/java/javaOO/localclasses.html)

```java
class OuterClass {
    public static void foo() {
        class LocalClass {
        }
        
        LocalClass localClass = new LocalClass();
    }
}
```

#### 匿名クラス(anonymous classes)

[Anonymous Classes (The Java™ Tutorials > Learning the Java Language > Classes and Objects)](https://docs.oracle.com/javase/tutorial/java/javaOO/anonymousclasses.html)

```java
class OuterClass {
    interface HelloWorld {
        public void greet();
    }
    
    public void hello() {
        HelloWorld japaneseGreeting = new HelloWorld() {
            public void greet() {
                System.out.println("こんにちは！");
            }
        }

        japaneseGreeting.greet();
    }
}
```

#### 入れ子クラスのメリット

https://docs.oracle.com/javase/tutorial/java/javaOO/nested.html  
> Why Use Nested Classes?

- 特定の場所で使われるクラスを論理的にグルーピングできる
- カプセル化を促進する
- 可読性とメンテナンス性が向上する

たしかに4種類の入れ子クラスは、上記3つのメリットをそれぞれ様々な粒度で提供しているように思える。


## 使いどころ - Builderパターン

入れ子クラスがどんなものなのか、というのと文法はわかったが、やはりもう少し実践的な例がないとイマイチ身になっている気がしない。そんな具合に少しモヤッていたところ、ちょうど読み始めた [Effective Java 第3版](https://www.amazon.co.jp/Effective-Java-%E7%AC%AC3%E7%89%88-Joshua-Bloch/dp/4621303252) に具体的かつ実践的な例が紹介されていた。



