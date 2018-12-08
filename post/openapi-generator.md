---
title: "OpenAPI Generator - community drivenで成長するコードジェネレータ"
date: 2018-05-12T23:13:49+09:00
draft: false
---

2018-05-12、[OpenAPI Generator](https://github.com/OpenAPITools/openapi-generator) が公開されました。

https://github.com/OpenAPITools/openapi-generator  

> OpenAPI Generator allows generation of API client libraries (SDK generation), server stubs, documentation and configuration automatically given an OpenAPI Spec

これは [Swagger Codegen](https://github.com/swagger-api/swagger-codegen) v2.4をフォークしたプロジェクトで、[OpenAPI](https://github.com/OAI/OpenAPI-Specification/)ドキュメントから様々なプログラミング言語のAPIクライアントやスタブサーバーなどのソースコードを生成するツールです。まだベータ版のような状態で、["v3.0.0"として初回リリースすることを予定しています](https://github.com/OpenAPITools/openapi-generator#11---compatibility) 。

私は "元" Swagger Codegen のコアメンバーであり、現在 OpenAPI Generator のコアメンバー/創立メンバーで少し中の事情に詳しいので、なぜフォークするに至ったのかといった経緯やOpenAPI Generatorについて書いていきたいと思います。  
なお、個人のブログに書いていることですので主観が入っている部分があるかもしれませんが大目に見ていただけますと幸いです。

<!--more-->

---

#### 経緯 - Swagger Codegenの開発状況

2017年7月に[OpenAPI Specification 3.0.0がリリース](https://www.openapis.org/blog/2017/07/26/the-oai-announces-the-openapi-specification-3-0-0)されてからもうじき1年が経ちますが、Swagger CodegenのOpenAPI Specification 3(以下、OAS3)対応はあまり進んでおらず喫緊の課題となっています。

Swagger Codegenの ["3.0.0" ブランチ](https://github.com/swagger-api/swagger-codegen/tree/3.0.0)で、OAS3への対応を目的とした開発をスポンサー企業であるSmartBear社のエンジニアの方が主導して進められていて、1月にリリース候補版(Swagger Codegen 3.0.0-rc0)がリリースされました。

[Release Swagger Codegen 3.0.0-rc0 has been released! · swagger-api/swagger-codegen](https://github.com/swagger-api/swagger-codegen/releases/tag/v3.0.0-rc0)  
なお、リリース候補版ではOAS3に対応している言語はまだごく一部のみとなっています。

このリリースには主目的であるOAS3の対応以外にも大きな変更が加えられており、内部のジェネレータ部分を[別リポジトリ](https://github.com/swagger-api/swagger-codegen-generators)に切り離したり、テンプレートエンジンを別のライブラリに切り替えています。

残念なことに、これらの変更は主導しているエンジニアが独断で行ったことです。コミュニティのリソースは有限であるため、私たちはOAS3の対応に集中した方が良いと考えていますがコミュニティの声は取り入られませんでした。
また、v3.0.0の開発が行われているブランチでは[テストコードが一気に無効化された](https://github.com/swagger-api/swagger-codegen/commit/952695086f071c299ed17fa0a5dff84f7c435152)場面もあり、(言い方が悪いかもしれませんが)やや乱暴な印象を受けるところもあります。

Swagger Codegenはこれまで多くの開発者からなる素晴らしいコミュニティに支えられて開発されてきました。
もちろん、これからもそうあるべきだと考えていますが実情とは大きな乖離があります。  
この状況を憂慮した [William](https://github.com/wing328/) (Swagger Codegenのトップコントリビュータ) はSmartBear社のエンジニアの方々と話し合いを重ねてきましたが、Williamの意向は受け入れられませんでした。

そこで、Williamが中心になってコアメンバーやテクニカルコミッティと話し合った結果、SmartBear社のエンジニアを説得するのは諦め、Swagger Codegenをフォークし、別のプロジェクトとしてこれまで通りのcommunity drivenな開発を続けることを決めました。

以上がSwagger Codegenをフォークするに至った経緯の大要となります。  

OpenAPI Generatorの [README](https://github.com/OpenAPITools/openapi-generator#63---history-of-openapi-generator) や [Q&A](https://github.com/OpenAPITools/openapi-generator/blob/master/docs/qna.md) にもそのあたりの記載がありますのでご興味があればぜひ御覧ください。

---

#### OpenAPI Generator

Swagger Codegen v3 及びその開発体制を批判することは当記事の目的ではありません。

OpenAPI Generatorは William の呼びかけで集まった[40人以上のコントリビュータ](https://github.com/OpenAPITools/openapi-generator/#founding-members-alphabetical-order)によって開発が進められ、本日の公開を迎えることができました。  
開発リソースをOAS3の対応に集中し、すべての言語で一通りの実装を行いました。
ですが、まだ不完全な箇所や各言語固有の課題などがまだまだ残っています。私たちが気づけていないジェネレータの不具合もあると思います。

私たちはみなさんからのフィードバック/コントリビュートを必要としています。  
ぜひOpenAPI Generatorでみなさんがお使いのプログラミング言語の[コードを生成してみてください](https://github.com/OpenAPITools/openapi-generator/#2---getting-started)。(APIクライアントだけでも[40以上の言語をサポートしています](https://github.com/OpenAPITools/openapi-generator/#overview))

一緒にOpenAPI Generatorを成長させませんか？

https://github.com/OpenAPITools/openapi-generator

OpenAPI Generator Core Team/Founding Member  
Akihito Nakano

---

#### 追記

プロジェクトの公開から半年以上が経ちました。[Publickey様に取り上げていただいた](https://www.publickey1.jp/blog/18/rest_apiapiopenapi_generatorswagger_generator.html)のを起点にして、国内でのOpenAPI Generatorの認知が確実に広がっているのを感じています。国内エンジニアのかたが[Gin(Golang)のジェネレータを追加](https://github.com/OpenAPITools/openapi-generator/pull/1048)したり、プルリクエストをされているのを頻繁に見かけるようになりました。🙏🙏🙏

このブログ記事も少しはアクセスがあるようですので、現時点(2018年12月)のOpenAPI Generatorの今後の展望を追記しておきます。

##### OpenAPI 3.0対応の拡充

すべての言語でOAS3に対応したと前述しましたが、とはいっても3.0で追加されたすべての機能を網羅できているわけではありません。たとえば[Callbackオブジェクト](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#callback-object)は、現在ジェネレータの[コア部分の対応](https://github.com/OpenAPITools/openapi-generator/pull/861)が済んだところで、各言語の対応はまだこれからという状況です。ですので短期的には、このような網羅できていない部分の対応を進めるのが課題の一つです(開発のロードマップを[こちら](https://github.com/OpenAPITools/openapi-generator/blob/master/docs/roadmap.adoc)で公開しています)。

##### より強いコミュニティへ

OpenAPI Generatorは「コミュニティ主体」のプロジェクトですので、よりたくさんのエンジニアに関わっていただきたいと考えています。現在、コミュニティには[Core Team](https://github.com/OpenAPITools/openapi-generator#61---openapi-generator-core-team)と[Technical Committee](https://github.com/OpenAPITools/openapi-generator#62---openapi-generator-technical-committee)という2つの役割を設けていますが、新しい役割を新設して、アクティブに活動されている方に積極的に権限を渡していく試みを始めています。このようにプロジェクトの横への広がりだけでなく、関心のある方はより深く関わっていけるよう縦の広がりも作っていくことを進めています。

もし、OpenAPI Generatorを触ってみて気になったことや不明点がありましたら[国内エンジニア向けの gitter チャットルーム](https://gitter.im/openapi-generator-japanese/Lobby)がありますのでぜひ覗いてみてください。日本国内のコントリビュータが集まっていますので、日本語で気軽に相談できます。
