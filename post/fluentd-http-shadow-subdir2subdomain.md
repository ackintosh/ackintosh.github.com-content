+++
date = "2017-08-15T16:39:59+09:00"
draft = false
title = "サブディレクトリからサブドメインに移行するコンテンツを fluent-plugin-http_shadow でテストする"
tags = ["fluentd"]
+++

(うまく伝わるタイトルが思いつかなかった...)

[fluent-plugin-http_shadow で会員向けコンテンツをテストする](https://ackintosh.github.io/blog/2017/07/30/fluentd-http-shadow/) に引き続き、[fluent-plugin-http_shadow](https://github.com/toyama0919/fluent-plugin-http_shadow) を使ったシャドウプロキシについて。

<!--more-->

### やりたいこと

本番のコンテンツは、サブディレクトリで別れている  
が  
リクエストの複製先はサブドメインで別れている場合。

- サブディレクトリで複製先を分けたい
- 複製したリクエストのパスからサブディレクトリを除きたい

| 本番 | テスト |
|:-----------|:------------|
| http://example.com/test1/awesome_content| http://test1.example.com/awesome_content |
| http://example.com/test2/awesome_content| http://test2.example.com/awesome_content |

（その他 (http://example.com/**) は、testx.example.com にパスをいじらずに送信する）

### 設定

```td-agent.conf
<source>
  type tail
  format apache
  tag apache.access
  ...
</source>

<match apache.access>
  @type copy
  <store>
    @type relabel
    @label @test1
  </store>
  <store>
    @type relabel
    @label @test2
  </store>
  <store>
    @type relabel
    @label @testx
  </store>
</match>

# /test1 -> test1.example.com
<label @test1>
  <filter>
    @type grep
    regexp1 path ^/test1
  </filter>
  <filter>
    @type record_transformer
    enable_ruby true
    <record>
      path ${record["path"].gsub(/\A\/test1\//, "/")}
    </record>
  </filter>
  <match **>
    type http_shadow
    host test1.example.com
    ...
  </match>
</label>

# /test2 -> test2.example.com
<label @test1>
  <filter>
    @type grep
    regexp1 path ^/test2
  </filter>
  <filter>
    @type record_transformer
    enable_ruby true
    <record>
      path ${record["path"].gsub(/\A\/test2\//, "/")}
    </record>
  </filter>
  <match **>
    type http_shadow
    host test2.example.com
    ...
  </match>
</label>

# その他 -> testx.example.com
<label @testx>
  <filter>
    @type grep
    exclude1 path ^/test
  </filter>
  <match **>
    type http_shadow
    host testx.example.com
    ...
  </match>
</label>

```

##### 補足

- grep プラグイン
  - v0.12.38 以降では `regexpN` `excludeN ` は非推奨なので代わりに `<regexp>` ディレクティブを使う
  - https://docs.fluentd.org/v0.12/articles/filter_grep#ltregexpgt-directive-optional


### 動作確認

##### /test1 -> test1.example.com

本番のアクセスログ

```
xxx.xxx.xxx.xxx - - [15/Aug/2017:09:47:08 +0000] "GET /test1/awesome_content/test1.html HTTP/1.1" 200 230 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36"
```

test1.example.com のアクセスログ

```
xxx.xxx.xxx.xxx - - [15/Aug/2017:09:47:10 +0000] "GET /awesome_content/test1.html HTTP/1.1" 200 504 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36"
```

##### その他 -> testx.example.com

本番のアクセスログ

```
xxx.xxx.xxx.xxx - - [15/Aug/2017:09:49:42 +0000] "GET /awesome_content/testx.html HTTP/1.1" 200 223 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36"
```

testx.example.com のアクセスログ

```
xxx.xxx.xxx.xxx - - [15/Aug/2017:09:49:42 +0000] "GET /awesome_content/testx.html HTTP/1.1" 200 505 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36" 
```