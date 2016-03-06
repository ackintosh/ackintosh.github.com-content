+++
date = "2012-07-10T16:59:25+09:00"
draft = false
title = "Markdown記法のパーサー　markdown-jsを使う"
tags = ["javascript"]
+++

![](https://dl.dropbox.com/u/22083548/octopress/markdown-to-html.jpeg)

最近githubやOctopressを使うようになってきたので、markdownをしっかり覚えたい今日この頃です。  
ふと、Javascriptでmarkdown→HTMLに変換してくれるのはないかなと気になったので
今回markdown-jsというのを使ってみました。

<!--more-->

*markdown-js*  
<a href="https://github.com/evilstreak/markdown-js" target="_blank">https://github.com/evilstreak/markdown-js</a>

#### 例
```javascript
var md = "#markdown";
console.dir(window.markdown.toHTML(md));
```

#### リアルタイムにmarkdown→HTMLに変換してみる
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>markdown 2 html</title>
    <script type="text/javascript" src="./lib/markdown.js"></script>
    <script type="text/javascript"
src="http://ajax.googleapis.com/ajax/libs/jquery/1.7.2/jquery.min.js"></script>
</head>
<body>
    <textarea id="markdown"></textarea>
    <div id="html"></div>
    <script type="text/javascript">
        $('#markdown').keyup(function (e){
        $('#html').html(window.markdown.toHTML($('#markdown').val()));
        });
    </script>
</body>
```

### 参考URL
にのせき日記  
<a href="http://d.hatena.ne.jp/ninoseki/20110620/1308574793" target="_blank">Javascript製Markdown記法パーサー、markdown-js</a>
