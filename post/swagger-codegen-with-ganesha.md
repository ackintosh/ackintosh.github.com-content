+++
date = "2017-04-09T16:33:27+09:00"
title = "Swagger Codegen + CircuitBreaker(Ganesha)"
draft = false
tags = ["php"]
+++
<!--more-->

## Swagger Codegen とは

[swagger-api/swagger-codegen](https://github.com/swagger-api/swagger-codegen)

OpenAPI / Swagger に沿った定義から色々な言語のクライアントやテスト、スタブを生成できるツール。  
Mac なら [Homebrew でさくっと試せる](https://github.com/swagger-api/swagger-codegen#homebrew)。

```
$ brew install swagger-codegen
$ swagger-codegen generate -i http://petstore.swagger.io/v2/swagger.json -l php -o .
$ tree
.
└── SwaggerClient-php
    ├── README.md
    ├── autoload.php
    ├── composer.json
    ├── docs
    │   ├── Api
    │   │   ├── PetApi.md
    │   │   ├── StoreApi.md
    │   │   └── UserApi.md
    │   └── Model
    │       ├── ApiResponse.md
    │       ├── Category.md
    │       ├── Order.md
    │       ├── Pet.md
    │       ├── Tag.md
    │       └── User.md
    ├── git_push.sh
    ├── lib
    │   ├── Api
    │   │   ├── PetApi.php
    │   │   ├── StoreApi.php
    │   │   └── UserApi.php
    │   ├── ApiClient.php
    │   ├── ApiException.php
    │   ├── Configuration.php
    │   ├── Model
    │   │   ├── ApiResponse.php
    │   │   ├── Category.php
    │   │   ├── Order.php
    │   │   ├── Pet.php
    │   │   ├── Tag.php
    │   │   └── User.php
    │   └── ObjectSerializer.php
    ├── phpunit.xml.dist
    └── test
        ├── Api
        │   ├── PetApiTest.php
        │   ├── StoreApiTest.php
        │   └── UserApiTest.php
        └── Model
            ├── ApiResponseTest.php
            ├── CategoryTest.php
            ├── OrderTest.php
            ├── PetTest.php
            ├── TagTest.php
            └── UserTest.php
```

とても良さそうですが、生成したコードがそのまま実際のサービスで使えるケースは少ないのではないでしょうか。たとえば外部 API の呼び出しでは、障害の連鎖を防ぐために [CircuitBreaker パターン](https://martinfowler.com/bliki/CircuitBreaker.html) の適用がとても有効です。

PHP の CircuitBreaker 実装 [Ganesha](https://github.com/ackintosh/ganesha/) の作者であるわたしとしては、Swagger Codegen で生成したクライアントコードに Ganesha を組み込む方法を明らかにしておきたいところです。


## CircuitBreaker(Ganesha)を組み込む

### オリジナルのテンプレートを用意する方法

Swagger Codegen は [こちら](https://github.com/swagger-api/swagger-codegen/tree/master/modules/swagger-codegen/src/main/resources/php) のディレクトリにあるテンプレートを元にしてコードを生成しますが、  
`-t` オプションでオリジナルのテンプレートを指定することもできます。

```
$ swagger-codegen help generate
...
...
        -t <template directory>, --template-dir <template directory>
            folder containing the template files
...
...
```


なので、composer.json と ApiClient.php の元になるテンプレートを適当なディレクトリにコピーして [こんな感じ](https://gist.github.com/ackintosh/b31193abc9a61f16dbb9f19652cc7215) でいじれば、Ganesha を利用したコードが生成されます。

```
$ swagger-codegen generate -i http://petstore.swagger.io/v2/swagger.json -l php -o . -t mytemplates
```

```diff
class ApiClient
    public function __construct(\Swagger\Client\Configuration $config = null)
    {
        if ($config === null) {
            $config = Configuration::getDefaultConfiguration();
        }

        $this->config = $config;
        $this->serializer = new ObjectSerializer();
+        $m = new \Memcached();
+        $m->addServer('localhost', 11211);
+        $this->ganesha = Builder::build([
+            'failureRate' => 50,
+            'adapter'     => new Ackintosh\Ganesha\Storage\Adapter\Memcached($m),
+        ]);
    }

    public function callApi($resourcePath, $method, $queryParams, $postData, $headerParams, $responseType = null, $endpointPath = null)
    {

+        if (!$ganesha->isAvailable($url)) {
+            throw new ApiException("$url is not available");
+        }
```

### ApiClient を継承したクラスを用意する方法

また、できるだけ独自のロジックを外出ししたい場合は [こちら](http://int128.hatenablog.com/entry/2017/03/02/003332) のように ApiClient を継承したクラスを用意すると良さそうです。

##### 参考
- [Swagger CodegenでPHPクライアントを生成する - GeekFactory](http://int128.hatenablog.com/entry/2017/03/02/003332)

> 今のところ、APIクライアントにはインターセプタの仕組みは用意されていないようです。共通処理を入れたい場合は ApiClient クラスを継承した独自クラスを定義し、APIクライアントのコンストラクタに渡すとよいでしょう。


ただ、この場合は "継承したクラスを必ず API クライアントのコンストラクタに渡さないといけない" ので、その責務を別のオブジェクトに持たせるためにファクトリや DI コンテナの必要性が出てくるのではないかと思っています。

## (追記1) Swagger Codegen ver2.3 からはクライアントとして Guzzle が使われている

Swagger Codegen トップコントリビューターの [@wing328](https://twitter.com/wing328) からリプライをいただいた。


<blockquote class="twitter-tweet" data-lang="ja"><p lang="en" dir="ltr"><a href="https://twitter.com/NAKANO_Akihito">@NAKANO_Akihito</a> Thx for writing about Swagger Codegen. You may want to checkout 2.3.0 branch to build the PHP API client, which uses Guzzle instead of curl</p>&mdash; wing328 (@wing328) <a href="https://twitter.com/wing328/status/851445713028304896">2017年4月10日</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

なので、今後は `オリジナルのテンプレートを用意する方法` ではなく  
`GuzzleHttp\Client` を継承(または `GuzzleHttp\ClientInterface` を実装)したクライアントを用意する方法が良さそう。

## (追記2) CircuitBreaker によって遮断された場合は別の例外を投げる

（ついでに追記）

[Release It! 本番用ソフトウェア製品の設計とデプロイのために](https://www.amazon.co.jp/Release-%E6%9C%AC%E7%95%AA%E7%94%A8%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E8%A3%BD%E5%93%81%E3%81%AE%E8%A8%AD%E8%A8%88%E3%81%A8%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%AE%E3%81%9F%E3%82%81%E3%81%AB-Michael-T-Nygard/dp/4274067491/ref=sr_1_1?s=books&ie=UTF8&qid=1491103513&sr=1-1&keywords=release+it) には CircuitBreaker の利用について下記のように書かれている。

<a href="https://www.amazon.co.jp/Release-%E6%9C%AC%E7%95%AA%E7%94%A8%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E8%A3%BD%E5%93%81%E3%81%AE%E8%A8%AD%E8%A8%88%E3%81%A8%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%AE%E3%81%9F%E3%82%81%E3%81%AB-Michael-T-Nygard/dp/4274067491/ref=as_li_ss_il?s=books&ie=UTF8&qid=1491103513&sr=1-1&keywords=release+it&linkCode=li2&tag=akihito0a-22&linkId=3225755b061ff6dc6147eeaa1b8cd8c3" target="_blank"><img border="0" src="//ws-fe.amazon-adsystem.com/widgets/q?_encoding=UTF8&ASIN=4274067491&Format=_SL160_&ID=AsinImage&MarketPlace=JP&ServiceVersion=20070822&WS=1&tag=akihito0a-22" ></a><img src="https://ir-jp.amazon-adsystem.com/e/ir?t=akihito0a-22&l=li2&o=9&a=4274067491" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />


> ユーザーに適切なフィードバックを提供するには、「開」のときには別な種類の例外を発生させると都合がよい。そうすれば、呼び出しをするコードでその種の例外を違ったやり方で処理できるようになる。

CircuitBreaker が組み込まれたクライアントを利用する側からしたら、例外が起きた時にそれが CircuitBreaker によって遮断されたものなのかどうかは重要。  
たとえば [決済処理において、リクエストが遮断された場合は復旧後に決済させる](/blog/2017/02/18/2017-02-18/) 仕組みをとっている事例があり、これは Release It! に書かれているとおり個別の例外を発生させないとできない。

[Yahoo! JAPAN MeetUp #9 (EC技術カンファレンス) ・ 暁](https://ackintosh.github.io/blog/2017/02/18/2017-02-18/)

> #### カード後決済
> - 決済サービスが障害で応答しない場合、まず受注してしまう
> - 復旧したあとでカード決済させる
> - サーキットブレーカーで遮断された決済を、カード後決済させる


なので (追記1) に書いた `GuzzleHttp\Client` を継承したクライアントを用意する際に、その辺の考慮も必要。
