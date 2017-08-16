+++
date = "2017-08-16T14:34:19+09:00"
draft = false
title = "memory_limit を超えないように HTTP リクエストのレスポンスを受取る"
tags = ["php"]
+++

<!--more-->

#### サーバー

```server.php
set_time_limit(0);

echo str_repeat('.', 1024 * 1024 * 256);
```

#### file_get_contents :ng:

```client.php
echo 'memory_limit: ' . ini_get('memory_limit') . "\n";

file_get_contents($url);

register_shutdown_function(function () {
    echo sprintf("\nmemory_get_peak_usage: %dMB\n", memory_get_peak_usage() / 1024 / 1024);
});

// memory_limit: 128M
// PHP Fatal error:  Allowed memory size of 134217728 bytes exhausted (tried to allocate 130023456 bytes) in /Users/akihito1/hoge.php on line 7
```

#### ストリームを使う :ok:

```client.php
echo 'memory_limit: ' . ini_get('memory_limit') . "\n";

$ch = curl_init($url);
$stream = fopen('php://temp', 'w+');
curl_setopt_array($ch, [
    CURLOPT_FILE => $stream,
]);
curl_exec($ch);
curl_close($ch);

rewind($stream);
while (!feof($stream)) {
    echo fread($stream, 10);
}
fclose($stream);

register_shutdown_function(function () {
    echo sprintf("\nmemory_get_peak_usage: %dMB\n", memory_get_peak_usage() / 1024 / 1024);
});

// memory_limit: 128M
// memory_get_peak_usage: 2MB
```

#### Guzzle を使う場合 :ok:

```client.php
$r = (new GuzzleHttp\Client)->get($url);
while (!$r->getBody()->eof()) {
    echo $r->getBody()->read(10);
}

// memory_limit: 128M
// memory_get_peak_usage: 3MB
```