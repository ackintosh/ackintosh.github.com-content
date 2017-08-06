+++
date = "2017-07-30T19:59:17+09:00"
draft = false
title = "fluent-plugin-http_shadow で会員向けコンテンツをテストする"
+++

- [toyama0919/fluent-plugin-http_shadow: copy http request. use shadow proxy server.](https://github.com/toyama0919/fluent-plugin-http_shadow)
- [fluentdで本番環境を再現する - Qiita](http://qiita.com/toyama0919/items/ae4bc88423317e6668b1)


Fluentd を使って ShadowProxy できるプラグイン。フロントに手を入れずに簡単・安全にできるのが魅力。現状、リクエストボディが送信できない(※)が、 GET アクセスが大部分を占めるようなロールであれば充分かなと。

<!--more-->

※ [Quipper版](https://github.com/quipper/fluent-plugin-http_shadow)ではリクエストボディを送信可能になっている

仕事で使おうと思っていじっていたので、ちょっとしたことだけどブログに書いておく。  
検討中のかたのご参考になれば幸い。  
( 以下、 php + apache )


### サンプルページ

- ログインフォームを表示
- ログイン後、`/?p=mypage` に遷移
- 未ログインで `/?p=mypage` にアクセスした場合、ログインフォームにリダイレクト


index.php

```php
<?php
if (
      !ini_set('session.save_handler', 'memcached')
   || !ini_set('session.save_path', 'memcacheサーバー')
   || !session_start()
){
    die('セッションを開始できません');
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $_SESSION['login'] = true;
    header('Location: /?p=mypage');
    exit;
}

if (isset($_GET['p']) && $_GET['p'] === 'mypage') {
    if (!isset($_SESSION['login'])) {
        header('Location: /');
        exit;
    }
    echo 'welcome';
    exit;
}

require_once('login.html');

```

login.html

```html
<html>
<head>
  <title>login</title>
</head>
<body>
  <form action="/" method="post">
    <input type="submit" name="login" value="login" />
  </form>
</body>
</html>

```

### apache のアクセスログに cookie を残す設定

- `%{クッキー名}C` で特定の cookie をログに残せる

httpd.conf

```httpd.conf
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" \"%{PHPSESSID}C\"" mylogformat
```

access.log

```
xxx.xxx.xxx.xxx - - [02/Aug/2017:23:54:34 +0000] "GET / HTTP/1.1" 200 159 "-" "UA" "br0sq9o7l72p7ioqfi5hmhg5c3"
```

- cookie が無い場合は `-`

access.log

```
xxx.xxx.xxx.xxx - - [02/Aug/2017:23:54:15 +0000] "GET / HTTP/1.1" 200 159 "-" "UA" "-"
```

### Fluentd の設定

- source ディレクティブ

cookie を扱うために format を調整

```
<source>
  type tail
  # apache2 フォーマットの末尾に session_id を追加
  # 参考 https://docs.fluentd.org/v0.12/articles/parser_apache2
  format /^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)" "(?<session_id>[^\"]*)")?$/
  time_format %d/%b/%Y:%H:%M:%S %z
  types code:integer, size:integer
  path /var/log/httpd/access_log
  pos_file /var/log/td-agent/access.pos
  tag apache.access
</source>
```

-  match ディレクティブ

`cookie_hash` で cookie を送信できる

```
<match apache.access>
  type http_shadow
  host 送信先
  path_format ${path}
  method_key method
  header_hash { "Referer": "${referer}", "User-Agent": "${agent}" }
  cookie_hash {"PHPSESSID": "${session_id}"}
  flush_interval 2s
  support_methods [ "get" ]
</match>
```


### 動作確認

- 元のサーバーにアクセス
- ログインボタンを押下

元のサーバーのアクセスログ

```
xxx.xxx.xxx.xxx - - [06/Aug/2017:04:16:32 +0000] "GET / HTTP/1.1" 200 159 "-" "UA" "57u32o65b5q6f6m1fit4onjq06"
xxx.xxx.xxx.xxx - - [06/Aug/2017:04:16:36 +0000] "POST / HTTP/1.1" 302 - "http://example.com/" "UA" "57u32o65b5q6f6m1fit4onjq06"
xxx.xxx.xxx.xxx - - [06/Aug/2017:04:16:36 +0000] "GET /?p=mypage HTTP/1.1" 200 7 "http://example.com/" "UA" "57u32o65b5q6f6m1fit4onjq06"
```

プロキシ先のアクセスログ

```
xxx.xxx.xxx.xxx - - [06/Aug/2017:04:16:32 +0000] "GET / HTTP/1.1" 200 442 "-" "UA" "57u32o65b5q6f6m1fit4onjq06"
xxx.xxx.xxx.xxx - - [06/Aug/2017:04:16:36 +0000] "GET /?p=mypage HTTP/1.1" 200 264 "http://example.com/" "UA" "57u32o65b5q6f6m1fit4onjq06"
```

##### ポイント

- プロキシ先にも cookie 付きでアクセスが来ていること
- プロキシ先の `GET /?p=mypage` のステータスコードが 200 であること
	- ちゃんと cookie が渡せてないと、未ログイン判定されて 302 になってしまう
- (補足) [先日だした PR](https://github.com/toyama0919/fluent-plugin-http_shadow/pull/4) を使っているので GET だけが送信されている

