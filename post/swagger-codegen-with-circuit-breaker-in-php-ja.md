---
title: "Swagger Codegen + Circuit Breaker in #PHP"
date: 2018-02-05T22:51:53+09:00
draft: false
---

( English version is coming soon! )

以前 [Swagger Codegen + CircuitBreaker(Ganesha)](/blog/2017/04/09/swagger-codegen-with-ganesha/) でSwagger Codegenと拙作の[Ganesha](https://github.com/ackintosh/ganesha) (Circuit BreakerのPHP実装)を組み合わせる方法を書いた。  
その後、Swagger CodegenとGaneshaの双方ともバージョンアップし親和性が高まり、よりシンプルな方法で組み込めるようになったので、改めてSwagger CodegenやCircuit Breakerの概要も含めてご紹介したい。

<!--more-->

## Swagger Codegenとは

https://github.com/swagger-api/swagger-codegen

OpenAPI仕様からドキュメント、APIクライアントやサーバースタブ等を自動生成するツールで、例えばAPIクライアントは当記事執筆時点で30以上の言語をサポートしており活発なコミュニティによってその数は増え続けている。

利用するには、ビルド済みのjarファイルをダウンロードしたり、Macユーザーならhomebrewで安定版をインストールして使うことができるが、下記ではリポジトリから最新のmasterブランチをcloneして使う方法を挙げておく。

```bash
# setup
$ git clone https://github.com/swagger-api/swagger-codegen
$ cd swagger-codegen
$ ./run-in-docker.sh mvn package

# generate php client
$ ./run-in-docker.sh generate \
    -i path/to/your-api-specification.yml \
    -l php \
    -o /gen/out/php-client
$ ls -la ./out/php-client
```

これだけでAPI仕様に沿ったPHPクライアントが生成できる。  
その他、そもそものOpenAPIの仕様や生成するコードをカスタマイズする方法についての詳細は、Swagger Codegenのトップコントリビュータ [@wing328](https://twitter.com/wing328) が執筆した電子書籍を参照することをお勧めします。

[A Beginner's Guide to Code Generation for REST APIs](https://gumroad.com/a/1072608371)

(なお現在、私を含むTechnical Committeeの日本人メンバーによって日本語訳が進められており、近日リリース予定です)

### PHPクライアントについて

Swagger Codegen v2.3からはHTTPクライアントとしてGuzzleを採用している。(それ以前はcurl関数をラップした独自のHTTPクライアントを使っていた)  
これにより非同期リクエストが可能になったり、Guzzleの [Middleware機構](http://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) を利用する各種ライブラリを組み込むことが可能になり利便性が一気に向上した。

[Packagistで "guzzle middleware" を検索した結果](https://packagist.org/?q=guzzle%20middleware&p=0)

後述するが、GaneshaもこのMiddleware機構を利用することでよりシンプルにSwagger Codegenと組み合わせることが出来るようになった。

## Circuit Breakerとは

https://martinfowler.com/bliki/CircuitBreaker.html

外部API呼び出しにおいて、呼び出し先が高負荷などにより応答しなくなった場合に、その障害が呼び出し元に連鎖してしまうことを防ぐための実装パターン。

Circuit BreakerがAPI呼び出しの成功/失敗(タイムアウト)を監視し、失敗が閾値を超えるとOpenステータスとなり、API呼び出しを遮断(Reject)する。この挙動により障害の連鎖を防止する。また、一定時間経過後にCircuit BreakerはHalf Openステータスとなり一部のAPI呼び出しを通すようになる。呼び出しが成功すれば障害が収束したとみなしてCircuit Breakerは元のステータス(Closed)となり、全ての呼び出しを許可する。

つまり人手を介することなくCircuit Breakerが呼び出し先の異常を検知し、飛び火することをいい感じに防止してくれる。Circuit Breakerのステータス遷移についてよりイメージしやすいように、Martin Fowler氏のブログから図を引用させていただく。

![circuit_breaker_state.png](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/swagger-codegen-with-circuit-breaker-in-php-ja/circuit_breaker_state.png)

([CircuitBreaker - martinfowler.com](https://martinfowler.com/bliki/CircuitBreaker.html)より引用)

Circuit BreakerのPHP実装はいくつかあるが、イチオシは拙作の [Ganesha](https://github.com/ackintosh/ganesha)。閾値の設定を [失敗率(Rate)や失敗数(Count)](https://github.com/ackintosh/ganesha#strategies) で設定でき、[Adapter機構](https://github.com/ackintosh/ganesha#adapters) で様々なストレージに対応している。  
ご興味があれば、(擬似的な)障害を起こしながらGaneshaの挙動を確認できる [サンプル](https://github.com/ackintosh/ganesha#are-you-interested) を用意しているので是非お試しいただきたい。

## Swagger Codegen + Ganesha

前置きが長くなってしまったが、ここまで読んでいただいたかたはSwagger CodegenでAPIクライアントを生成することの手軽さと、安定したサービス運用のためのCircuit Breakerの必要性をご理解いただけたのではないだろうか。以下、Swagger CodegenとGaneshaを組み合わせる方法をご紹介する。

まずはGaneshaが提供するGuzzle Middlewareを、Guzzleに組み込む。

```php
use Ackintosh\Ganesha\Builder;
use Ackintosh\Ganesha\GuzzleMiddleware;
use Ackintosh\Ganesha\Exception\RejectedException;
use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;

$ganesha = Builder::build([
    'timeWindow'           => 30,
    'failureRateThreshold' => 50,
    'minimumRequests'      => 10,
    'intervalToHalfOpen'   => 5,
    'adapter'              => $adapter,
]);
$middleware = new GuzzleMiddleware($ganesha);

$handlers = HandlerStack::create();
$handlers->push($middleware);

$client = new Client(['handler' => $handlers]);
```

あとは、そのHTTPクライアントをSwagger Codegenで生成したAPIクラスのコンストラクタに渡すだけ。

```php
$api = new PetApi($client);
```

これで、前述したとおりのロジックでGaneshaがAPI呼び出しを監視し、異常を検知すればリクエストを遮断し、自動的に復旧の判断もしてくれる。

### リクエストが遮断(Reject)されたときは？

Ganeshaが呼び出し先APIの異常を検知しリクエストを遮断した場合、例外 `RejectedException` を投げる。

```php
use Ackintosh\Ganesha\Exception\RejectedException;

try {
    $api->getPetById(123);
} catch (RejectedException $e) {
    awesomeErrorHandling($e);
}
```

この挙動は [Release It! 本番用ソフトウェア製品の設計とデプロイのために](https://www.amazon.co.jp/Release-%E6%9C%AC%E7%95%AA%E7%94%A8%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E8%A3%BD%E5%93%81%E3%81%AE%E8%A8%AD%E8%A8%88%E3%81%A8%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%AE%E3%81%9F%E3%82%81%E3%81%AB-Michael-T-Nygard/dp/4274067491/ref=sr_1_1?s=books&ie=UTF8&qid=1491103513&sr=1-1&keywords=release+it) の `5.2 ブレーカー` で説明されている下記に従っている。

> ブレーカーが「開」だと、すべての呼び出しが即座に失敗する。これは何らかの例外によって示すべきだろう。ユーザーに適切なフィードバックを提供するには、「開」のときには別な種類の例外を発生させると都合がよい。そうすれば、呼び出しをするコードでその種の例外を違ったやり方で処理できるようになる。
  
(P.101 5.2ブレーカー より引用)

### Ganeshaが利用するストレージで障害が起きた場合は？

GaneshaはAPI呼び出しの成功/失敗等をストレージに記録するが、そのストレージで障害が起きた場合、例外やエラーは発生させずリクエストを許可する。  
これはFail-silentに従った挙動で、例えば自動車設計において安全装置にトラブルが起きた場合、安全な状態を保つために通常はその機能を停止する方向に設計する。同様に、API呼び出しにおける安全装置であるCircuit Breakerの障害時はその機能を停止(例外等を発生させずリクエストを許可)するよう実装している。

代わりに、下記のようにストレージ障害にフックして任意の処理を行うことができる。

```php
$ganesha->subscribe(function ($event, $service, $message) {
    switch ($event) {
        case Ganesha::EVENT_STORAGE_ERROR:
            \YourMonitoringSystem::error($message);
            break;
        default:
            break;
    }
});
```

## まとめ

Swagger CodegenによるAPIクライアントの自動生成、Circuit Breakerによる障害連鎖の防止、それからCircuit BreakerのPHP実装であるGaneshaをSwagger Codegenで生成したAPIクライアントに組み込む方法を駆け足でご紹介した。

ご覧頂いたとおりGaneshaを組み込む方法はとてもシンプルなので、すでにSwagger Codegenでクライアントを生成して運用されている場合でも簡単に導入できる。  
Ganeshaについての詳細はぜひ [README](https://github.com/ackintosh/ganesha#ganesha) をご参照いただきたい。ご要望やPRも大歓迎なのでお気軽に。

---

2014年にMartin Fowler氏のブログ [Microservices](https://martinfowler.com/articles/microservices.html) で提唱されたマイクロサービスは、今日では広く知られ一般的になった手法だと思います。また、マイクロサービスがバズる以前から、コンポーネント同士をAPIで繋いでサービスを構成しているケースは多くあったのではないでしょうか。  
みなさんは障害の連鎖を経験されたことがありますか？恥ずかしながら私はあります(一度ではなく、何度も...)。あのときの悔しさがGaneshaを開発するモチベーションになっているのかもしれません。

この記事がみなさまの開発効率化とサービス安定運用のご参考になれば幸いです。