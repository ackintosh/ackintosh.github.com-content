+++
date = "2018-12-08T13:42:50+09:00"
draft = false
title = "Java8で開発しながら、Java9で非推奨になった構文が使われているかチェックする"
tags = []
image = ""
comments = false	# set false to hide Disqus
share = false	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = ""
+++

## 背景

[OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator)は現在 [Java8 での動作](https://github.com/OpenAPITools/openapi-generator#14---build-projects) が基本になっているが、一方で下記のPR/issueのようにJDK9以降のサポートを進めていく流れがある。

- [Add JDK 9 support by wing328 · Pull Request #1188 · OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator/pull/1188)
- [Investigate Java 9+ support · Issue #263 · OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator/issues/263)
- [Update brew formula to support JDK9 · Issue #1203 · OpenAPITools/openapi-generator](https://github.com/OpenAPITools/openapi-generator/issues/1203)

自分もこの辺に絡んでいきたい気持ちになってきたので、Java8で開発しながら、Java9(またはそれ以降のバージョン)で非推奨になった構文が手元のソースコードで使われているかチェックする方法が知りたくなった。

<!--more-->

##### Twitterでつぶやいたら返信をいただけた

なにぶんJavaに慣れていないのでどうしたら良いかわからないので、Twitterでつぶやいたら返信をいただけた 🙏 感謝!!


<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">情報源としてはJava9のリリースノートに<br>Removed APIs, Features, and Options<a href="https://t.co/1I8Z7HvuxY">https://t.co/1I8Z7HvuxY</a><br>Deprecated APIs, Features, and Options<a href="https://t.co/n1UbBwTn1W">https://t.co/n1UbBwTn1W</a><br>というものがあります <a href="https://t.co/9ndZ92mjPb">https://t.co/9ndZ92mjPb</a></p>&mdash; なぎせ ゆうき (@nagise) <a href="https://twitter.com/nagise/status/1071248156199870464?ref_src=twsrc%5Etfw">2018年12月8日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">とはいえ、コード中に何が含まれているかを網羅的にチェックするのは難しいので、結局Java9をインストールしてjavacで警告が出るものを探す、というのが効率的ではないかと思います</p>&mdash; なぎせ ゆうき (@nagise) <a href="https://twitter.com/nagise/status/1071248469682057218?ref_src=twsrc%5Etfw">2018年12月8日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ふむふむ！

## 自分なりにやったことまとめ

以下、いただいたアドバイスをもとに自分なりにやってみたことをまとめる 📝

### サンプルコード

Java9から [Class.newInstance()が非推奨になった](https://docs.oracle.com/javase/9/docs/api/java/lang/Class.html#newInstance--) ので、これを警告してもらえることをゴールにする。

```java
package io.github.ackintosh;

public class Main {
    public static void main(String[] args) {
        try {
            System.out.println(String.class.newInstance());
        } catch (Exception e) {
            System.out.println(e);
        }
    }
}
```

Java8 では何事もなくコンパイルが終わる。

```bash
$ java -version
java version "1.8.0_65"

$ mvn clean compile
[INFO] Scanning for projects...
[INFO]
[INFO] --------------------< io.github.ackintosh:linttest >--------------------
[INFO] Building linttest 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ linttest ---
[INFO] Deleting /Users/akihito1/dev/linttest/target
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ linttest ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 0 resource
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ linttest ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/akihito1/dev/linttest/target/classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.484 s
[INFO] Finished at: 2018-12-08T14:10:41+09:00
[INFO] ------------------------------------------------------------------------
```

### コンパイル時に引数を渡す

[Apache Maven Compiler Plugin](https://maven.apache.org/plugins/maven-compiler-plugin/)でコンパイル時に引数を渡せるようにする。

pom.xml

```xml
            ...
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <compilerArgument>${compilerArgument}</compilerArgument>
                </configuration>
            </plugin>
            ...
```

### JDK9でコンパイルする

[MavenのDockerイメージ](https://hub.docker.com/_/maven/)を使う。


```bash
$ docker run --rm -v (PWD):/lint -w /lint maven:3-jdk-9 mvn clean compile -DcompilerArgument=-Xlint:deprecation
...
...
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /lint/target/classes
[WARNING] /lint/src/main/java/io/github/ackintosh/Main.java:[7,44] newInstance() in java.lang.Class has been deprecated
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 49.018 s
[INFO] Finished at: 2018-12-08T05:17:09Z
[INFO] ------------------------------------------------------------------------
```

> [WARNING] /lint/src/main/java/io/github/ackintosh/Main.java:[7,44] newInstance() in java.lang.Class has been deprecated

警告が出た！👌

## 終わりに

`背景` に書いたが、なぜこれがやりたかったかというと[OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator)でJava9以降で非推奨とされている構文がどれくらい使われているかを把握するため。なので、OpenAPI Generatorで上記を試したら警告がたくさん出てきた！！

ということで、[OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator)にコントリビュートしてみたいというかたは、ネタはありますのでぜひお願いします!!!!1 (切実)
