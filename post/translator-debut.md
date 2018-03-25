---
title: "翻訳者デビュー - REST APIのためのコード生成入門"
date: 2018-03-25T12:13:22+09:00
draft: false
---

2018-03-16、Swagger Codegenについての電子書籍が販売開始した。

[REST APIのためのコード生成入門 - Swagger Codegenを利用したRESTful API開発の効率化](https://gumroad.com/l/swagger_codegen_beginner_jp)

<a href="https://gumroad.com/l/swagger_codegen_beginner_jp" target="_blank">
![REST APIのためのコード生成入門](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/translator-debut/swagger_codegen_beginner_jp.jpg)
</a>

内容はOpanAPIやSwagger Codegenの解説、導入〜自動生成するコードをカスタマイズする方法の紹介。ご興味があるかたは[1章まで無料で読める](https://drive.google.com/file/d/1bAKOvNY9y2QUJhRdrNbCfXqQxVXds6O3/view)ので是非ご覧いただきたい。

<!--more-->

原著はSwagger Codegenのトップコントリビュータである[@wing328](https://twitter.com/wing328)による [A Beginner's Guide to Code Generation for REST APIs](https://gumroad.com/l/swagger_codegen_beginner?locale=en) で、昨年末から[@taxpon](https://twitter.com/taxpon)さんと2人で翻訳作業をしていた。

[@taxpon](https://twitter.com/taxpon)さんのブログで翻訳に至った経緯、作業プロセスやSwagger Codegenの今後にも触れられているのでご参照いただきたい。

[Takuro Wada | Swagger Codegenに関する電子書籍の販売を開始しました](https://takuro.ws/2018/03/17/swagger-codegen-ebook/)


以下は、今回はじめて翻訳に挑戦してみた感想。

### 英語に全く自信が無かったが意外と出来た

今回の日本語訳の話をいただく前の2017年9月頃から、Swagger CodegenのTechnical Committee(PHP)としてissueやPRのやりとりをしていたが、英語が出来なさすぎてissueの返信に1時間以上かかることもよくあった。なので日本語訳の話をいただいた時はチャンスをもらえて嬉しい反面、とんでもなく巨大な壁が目の前に立ちはだかったように感じて "やりきれるのだろうか" と不安もかなりあった。

結果、2017年11月の作業開始から4ヶ月くらいで販売開始することができた。2018年1月の時点で翻訳自体はほぼできていて、それから先は調整(Swagger Codegenのバージョンアップに伴う内容の調整等)やレビューが主だったので、当初感じていた不安と比べたらだいぶ良いペースで進んでいた。


### 必要なのは日本語力だった

翻訳作業中にしばしば悩んでいたのが意訳の難しさだった。

「原著の意図は読み取れる。ただ直訳すると○○○なんだけど、もっと伝わりやすいイイ感じの日本語に意訳したいが、それが思いつかない。」というところで手が止まってしまうことが多かった。

例えば `1.3 Swagger Codegenのターゲット` で、APIのエバンジェリストがSwagger Codegenを利用するメリットを挙げている箇所で原著は、

```
The success of an API evangelist hinges on many factors and one of them is how many developers actually use the API in production.
```

となっていて、これを直訳すると

```
APIエバンジェリストの成功は多くの要因に左右され、そのうちの1つはAPIを実際にプロダクションで使用する開発者の数です。
```

になると思うが、日本語版ではこれを下記にしている。

```
APIをプロダクトで利用する開発者の数は、そのAPIの成功を示す重要なファクターのひとつです。
```

(ここだけ抜き出して例示しても、良くなっているかどうかわかりにくいかもしれない...)

### おわりに

今でも英語のやりとりには四苦八苦してはいるが、翻訳を経験したことで心理的ハードルはだいぶ下がったように感じている。また月並みだが、英語のドキュメントを眺めて内容を把握するのと、それを母国語で相手(読者のかた)に伝えるのは違う難しさがあって面白かった。

---

原著者であり、Swagger Codegenのトップコントリビュータである[@wing328](https://twitter.com/wing328)はコミュニティを本当に大切にしていて、彼のコミュニティに敬意を払う姿勢が本書の端々にもあらわれている。電子書籍の販売についても、販売によって得られるものをコアメンバーやテクニカルコミッティに還元したいという想いもあるようだ。

自分がSwagger Codegenに関わるようになった経緯は [Swagger Codegen Core Teamにジョインした](https://ackintosh.github.io/blog/2018/02/25/joined-to-swagger-codegen-core-team/) に書いたが、ちょっとしたきっかけでコントリビュートするようになり、それが今回日本語訳の話につながり貴重な経験をさせていただいた。これからもコミュニティの一員としてSwagger Codegenに貢献していきたい。

また、"Swagger Codegenへのコントリビュートに興味があるけど、何からやったらいいのかわからない" というかたは是非 [help wantedラベルが付いているissue](https://github.com/swagger-api/swagger-codegen/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22) を眺めてみることをおすすめします。または僕にご連絡ください :)